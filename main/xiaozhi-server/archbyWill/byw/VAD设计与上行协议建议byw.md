# 端侧 VAD 与 AI 模型侧 VAD：定位区别与通用上行帧协议建议

本文总结端侧 MCU 上的 VAD 与服务器/AI 模型侧 VAD 在定位、能力与工程落地上的区别，并给出一套与芯片无关（MCU-agnostic）的上行音频帧协议建议，适配 ESP、杰理等多家芯片。

## 1. 概要
- 端侧（MCU）VAD：低算力、低功耗、快速响应，用于本地门控、省带宽、配合唤醒词与采集起止控制。
- 服务器/AI 侧 VAD（如 Silero）：高准确率、鲁棒分段、阈值可调，服务 ASR 的精细端点检测、分句与打断控制。
- 最佳实践：级联/协同。MCU 做“粗门控 + 预滚缓存 + 起止标记”，服务器做“精细分段 + 一致化策略”。

## 2. 端侧 VAD vs 服务器/AI 侧 VAD
- 目标侧重
  - MCU：省电、省带宽、快速起停；在强噪和设备差异下尽量稳定。
  - 服务器：统一高质量分段，为 ASR/对话状态机提供可靠边界与打断能力。
- 算法与实现
  - MCU：能量/谱熵/零交叉等传统方法或轻量模型，定点/低内存实现。
  - 服务器：DNN/轻量深度模型输出逐帧概率，阈值/滞后可动态调参。
- 优势与风险
  - MCU：实时、低功耗；但更易误切起始/尾部，在远场/噪声下误报/漏报较多。
  - 服务器：准确率与一致性高；但若单独使用则需要持续上行音频，带宽/功耗更大。
- 在系统中的角色分工
  - MCU：入口守门员（是否上传、何时开始/结束、是否命中唤醒）。
  - 服务器：分段裁判员（如何切分、如何与 ASR/LLM 时序对齐与打断）。

## 3. 协同策略（推荐）
1) MCU 端开启 VAD 仅做"粗门控/起止标记"，并保留 200–500ms 预滚缓存（pre-roll）；检测到起始后携带缓存一起上报，避免切掉起始辅音。
2) 上行以固定帧长（10–30ms）持续推送，直到 MCU 判定结束并经历 hangover（200–500ms）。
3) 服务器端仍运行自身 VAD（如 Silero）做"精细分段/端点确认"，把 MCU 的 vad 标记作为提示而非唯一依据。
4) 如需极低时延的打断 TTS，可让服务器 VAD 的阈值略激进并在状态机中优先处理说话活动。

### 详细说明

#### 问题背景
- 如果只用MCU的VAD：容易误切语音开头或结尾，在噪音环境下不稳定
- 如果只用服务器VAD：MCU需要持续上传音频，耗电耗带宽
- 协同策略：让两者各司其职，互相补充。由于MCU的VAD在噪
  音环境下可能不稳定，服务器不完全信任MCU的VAD判断

#### 1) MCU端：粗门控 + 预滚缓存
```
正常状态：MCU不上传音频（省电省带宽）
检测到语音：开始录音，但不是从检测点开始！
             而是从200-500ms之前开始（预滚缓存）
             
举例：
时间轴：  |----静音----|语音开始|----说话中----|语音结束|----静音----|
MCU检测：                ↑这里检测到
实际发送：        ↑从这里开始发送（包含前面的缓存）
```

**为什么要预滚缓存？**
- 语音开头的"你好"中的"你"字可能被MCU VAD漏掉
- 有了缓存，即使MCU检测晚了，服务器也能收到完整的"你好"

#### 2) 持续推送直到结束
```
MCU检测到语音后：
- 每10-30ms发送一个音频帧
- 即使中间有短暂停顿也继续发送
- 直到确认语音真正结束（hangover时间后）
```

#### 3) 服务器端：精细分段
```
服务器收到音频流后：
- 不完全相信MCU的判断
- 用自己的VAD（如Silero）重新分析
- MCU说"开始"只是参考，服务器可能判断实际开始点不同
- MCU说"结束"也只是参考，服务器做最终切分决定
```

## 4. MCU 无关的能力协商（Hello/握手）
端侧在 hello 消息中声明能力，服务器据此选择策略（阈值、是否信任端侧起止等）。建议字段：
- audio.codec: opus | pcm16 | speex | g711u ...
- audio.sample_rate: 16000/8000/24000/48000
- audio.channels: 1/2
- audio.frame_ms: 10/20/30（单帧时长）
- vad.on_device: true/false
- vad.pre_roll_ms: N（若支持预滚缓存）
- dsp: ns=true/false, aec=true/false, agc=true/false
- kws.on_device: true/false
- transport: chunking=true/false, max_chunk_ms=N
- timing: device_ts=true/false（是否提供设备时戳）

### 详细说明

#### 问题背景
不同的MCU芝片能力差异很大：
- ESP32：有VAD、有回声消除、支持Opus编码
- 某便宜芝片：只能录PCM、没有VAD、没有降噪
- 杰理芝片：有自己的VAD算法、支持不同采样率

#### 解决方案：握手时告诉服务器“我能干什么”

##### 握手消息示例
```json
{
  "type": "hello",
  "device_id": "ESP32-001",
  "capabilities": {
    "audio": {
      "codec": "opus",           // 我支持Opus编码
      "sample_rate": 16000,      // 我用16kHz采样
      "channels": 1,             // 单声道
      "frame_ms": 20             // 每20ms发一帧
    },
    "vad": {
      "on_device": true,         // 我有VAD功能
      "pre_roll_ms": 300         // 我能提供300ms预滚缓存
    },
    "dsp": {
      "ns": true,                // 我有降噪
      "aec": true,               // 我有回声消除  
      "agc": false               // 我没有自动增益控制
    },
    "kws": {
      "on_device": true          // 我有唤醒词检测
    },
    "timing": {
      "device_ts": true          // 我能提供时间戋
    }
  }
}
```

#### 服务器根据能力调整策略

**场景1：高端MCU（如ESP32）**
```python
if capabilities["vad"]["on_device"]:
    # MCU有VAD，可以信任其起止标记
    server_vad_threshold = 0.3  # 阈值放松
    trust_device_segmentation = True
    
if capabilities["dsp"]["aec"]:
    # MCU已做回声消除，服务器端可以放宽要求
    audio_quality_expectation = "high"
```

**场景2：简单MCU**
```python
if not capabilities["vad"]["on_device"]:
    # MCU没有VAD，服务器要全权负责
    server_vad_threshold = 0.5  # 阈值提高
    trust_device_segmentation = False
    continuous_upload_mode = True  # 要求持续上传
    
if capabilities["audio"]["codec"] == "pcm16":
    # 只支持PCM，带宽会更大
    prepare_for_higher_bandwidth = True
```

#### 实际应用举例

**ESP32设备连接时：**
```
1. ESP32发送hello：我有VAD、支持Opus、有降噪
2. 服务器回应：好的，你可以用VAD控制上传，我相信你的判断
3. 通话中：ESP32检测到语音才开始上传，节省带宽
```

**简单MCU连接时：**
```
1. 简单MCU发送hello：我只能录PCM，没有VAD
2. 服务器回应：好的，你持续上传，我来做所有判断
3. 通话中：MCU一直录音上传，服务器用Silero VAD切分
```

这样的设计让同一个服务器可以适配各种不同能力的MCU设备，而不需要为每种芯片写专门的代码。

## 5. 上行二进制帧结构（统一协议，MCU 无关）
帧 = 固定头(小端) + 可选 TLV 扩展 + 音频负载。

- 固定头（建议字段，总计约 24 字节）：
  - proto_ver: uint8（协议版本，例 1）
  - codec: uint8（0=PCM16, 1=Opus, 2=Speex, 3=G711u...）
  - sample_rate: uint16（16000/8000/24000/48000）
  - channels: uint8（1/2）
  - frame_ms: uint8（10/20/30）
  - segment_id: uint32（一次语音段 ID，端侧递增）
  - chunk_id: uint32（段内块序号，从 0 递增）
  - flags: uint16（位图）
    - bit0: vad_active（此块包含语音）
    - bit1: vad_start（段起点）
    - bit2: vad_end（段终点，含 hangover）
    - bit3: kws_hit（可选：唤醒词命中）
    - bit4: aec_on
    - bit5: ns_on
    - bit6: agc_on
  - device_ts_ms: uint32（设备毫秒时戳，允许循环）
  - payload_len: uint16（负载字节数）

- 可选 TLV 扩展（按需）：
  - type=1: pre_roll_ms, uint16
  - type=2: gain_db_q8, int16（AGC/固定增益）
  - type=3: snr_q8, int16（估计信噪比）
  - type=4: rms_q8, uint16（电平）
  - type=5: latency_ms, uint16（端侧采集→打包延迟）

- 音频负载：
  - codec=PCM16：小端 int16，长度由 frame_ms × sample_rate × channels 决定。
  - codec=Opus：单个 Opus 包（无需再分片；如有分片用 chunk_id 表示顺序）。

## 6. 端侧状态机与时序建议
- 状态：idle → speech_active → hangover → idle。
- 参数范围（端侧可自定，服务器只读取元数据）：
  - frame_ms：10 或 20（10ms 低时延，20ms 带宽更优）
  - hangover_ms：200–500（端侧略长以避免短停顿误切）
  - pre_roll_ms：200–500（建议开启）
  - vad_start：连续 N 帧 active（N=2–4）
  - vad_end：连续 M 帧 inactive（M=10–25，依 frame_ms）
- 发送策略：
  - idle：不发音频或仅心跳。
  - speech_active：持续发送音频块（vad_active=1，起始块带 vad_start）。
  - hangover：继续发送若干 inactive 块补尾（末块带 vad_end）。
  - 每个段 segment_id+1，chunk_id 从 0 递增。

## 7. 服务器侧处理要点
- 仍运行服务器 VAD 做“精细分段/端点确认”，融合 MCU 的 vad 标记作为提示。
- 重组与对齐：按 segment_id、chunk_id 重组；vad_start/vad_end 作为边界初判；若有 pre_roll_ms，首块按此补齐窗口。
- 自适应：
  - 若 vad.on_device=false → 服务器全权分段。
  - 若端侧 VAD 误差较大 → 降低对 vad 标记权重，增大服务器 VAD 作用。
- 打断（barge-in）：服务器 VAD 阈值可略激进，优先于 ASR 执行打断策略。

## 8. 参数与工程建议（推荐范围，非强制）
- 编码：首选 Opus/16k/mono/frame=20ms；调试可用 PCM16。
- 码率：12–24 kbps（追求极低时延选 16–20kbps）。
- 采样率统一：16kHz，保证 Opus 编解码一致。
- 噪声处理：若 MCU 已做 AEC/NS → 服务器 VAD 阈值可放宽；否则适度提高。
- 时戳：无需 NTP 对齐，只需单调一致，便于服务器重排/对齐。

## 9. 不同 MCU 的映射示例
- ESP32（ESP-SR/AFE）：
  - 用 AFE 的 VAD 输出 → flags.vad_active；WakeNet 命中 → flags.kws_hit。
  - AEC/NS/AGC 状态映射至 flags。
  - 若支持环形缓冲，导出 pre_roll_ms；否则省略该 TLV。
- 杰理/恒玄/其它：
  - 使用各自 SDK 的 VAD/NS/AEC 输出，按同语义设置 flags。
  - 若无预滚缓存或设备时戳功能，对应字段留空/默认。
- 无 VAD 的设备：
  - hello 中声明 vad.on_device=false，持续流式上传，服务器全权做分段。

## 10. 版本与兼容
- proto_ver 从 1 起；新增字段通过 TLV 扩展，保持向后兼容。
- 服务器按 hello 能力启用/禁用解析分支与策略。
- 文本控制面可选事件：
  - { "type": "vad", "state": "start|end", "segment_id": N }
  - { "type": "kws", "word": "xiaozhi", "score": 0.87, "segment_id": N }

---

以上规范不绑定具体芯片，实现侧只需把各家 SDK 的 VAD/AEC/NS/KWS 状态映射到统一元数据与帧头字段即可。需要进一步示例（C 结构体定义或 Python 解析样例）时，可在此文档基础上追加附录。
