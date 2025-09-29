# 端侧 VAD 与服务器 VAD 协同设计（xiaozhi-server 实现说明）

本文围绕当前 `xiaozhi-server` 代码实现（2025Q3 版本）说明端侧设备与服务器分别承担的 VAD（Voice Activity Detection，语音活动检测）职责，并解答“为什么两端都需要 VAD”这一问题。核心思路是：服务器默认运行高精度 VAD 完成语音段切分，而端侧 VAD 通过控制消息参与门控和唤醒，让链路既节省资源又具备快速响应能力。

## 1. 为什么需要双端 VAD
- **省电与带宽**：端侧若能在本地判断静音，就可以在用户未说话时不上传音频，也能在命中唤醒词时快速上报，避免 24×7 持续推流。
- **精度与一致性**：服务器侧运行 Silero VAD（`core/providers/vad/silero.py`），结合滑动窗口、多阈值策略，可提供更稳定的起止定位，用于驱动 ASR、对话状态机以及 TTS 打断。
- **互为冗余**：噪声、远场等场景可能导致端侧误判；若仅依赖端侧，则容易切掉辅音或尾音；若仅依赖服务器，则端侧无法根据低功耗策略按需上传。双端配合可以把“门控”与“精细分段”拆开完成。

## 2. 服务器侧实现概览
整个处理链在 `core/connection.py`、`core/handle/receiveAudioHandle.py` 和 `core/providers/asr/base.py` 中完成：

1. **音频接入**：WebSocket/MQTT 收到的二进制音频帧被直接写入 `conn.asr_audio_queue`，没有额外自定义帧头，默认格式为 Opus 单包（`hello` 阶段可以声明其他格式）。
2. **Silero VAD 判定**：`SileroVAD.is_vad()` 会解码 Opus 为 PCM，放入 `conn.client_audio_buffer`，以 512 采样点为窗滑动估计语音概率，并结合上下阈值（默认 0.5/0.2）与滑动窗口计数，更新：
   - `conn.client_have_voice`：当前是否检测到语音；
   - `conn.client_voice_stop`：静音持续时间是否超过 `min_silence_duration_ms`，用于触发段落结尾；
   - `conn.last_activity_time`：最近语音活动时间戳。
3. **ASR 拼段**：`ASRProviderBase.receive_audio()` 将所有帧累积在 `conn.asr_audio`。当 `client_voice_stop` 为真时：
   - 复制并清空缓存；
   - 调用 `decode_opus()` 拼接 PCM；
   - 启动 ASR 推理（以及可选声纹识别），然后把文本发送到对话流程。
4. **超时与打断**：服务器根据 `last_activity_time` 自动检测长时间静音，必要时发送结束提示或关闭连接；若判定用户开口同时 TTS 在播报，会调用 `handleAbortMessage` 打断播报。

> 小结：服务器默认始终运行自身 VAD，保证语音段划分、TTS 打断和会话状态机的准确性，即使端侧完全不做 VAD 也能工作。

## 3. 端侧 VAD 的接入方式
端侧如果具备 VAD/唤醒能力，可通过文本控制信号与服务器协同。相关逻辑集中在 `core/handle/textHandler/listenMessageHandler.py`。

### 3.1 `listen` 控制消息
端侧通过 WebSocket 发送 JSON 消息：
```json
{
  "type": "listen",
  "mode": "auto|realtime|manual",
  "state": "start|stop|detect",
  "text": "可选，state=detect 时携带预判文本"
}
```
- `mode`
  - `auto`（默认）或 `realtime`：服务器以自身 VAD 结果为准，端侧消息仅作提示。
  - 其他取值（如 `manual`）：服务器转而使用端侧的 `state` 作为语音起止信号。
- `state`
  - `start`：端侧确认进入语音，服务器立即将 `client_have_voice` 设为真。
  - `stop`：端侧确认语音结束，标记 `client_voice_stop = True` 并触发一次 `handleAudioMessage(conn, b"")`，促使服务器尽快执行 ASR。
  - `detect`：端侧只是通报当前未检测到语音，同时清空服务器上的缓冲；若携带 `text`，服务器会把该文本直接送入对话流程（常用于端侧已完成 ASR 的模式）。

### 3.2 唤醒词与 Just Woken Up
当端侧在 `detect` 消息中传入唤醒词文本且命中配置，服务器会设置 `conn.just_woken_up = True`。此后第一帧音频会被忽略，待 `resume_vad_detection()` 延迟 1 秒后恢复，避免唤醒波段被误判为语音段。该逻辑在 `ListenTextMessageHandler` 与 `handleHelloMessage` 中共同作用。

### 3.3 没有端侧 VAD 的情况
- 端侧无需发送 `listen` 控制，持续推送音频即可。
- 服务器 Silero VAD 会自动完成分段；`client_listen_mode` 维持默认 `auto`。
- 断线/静音超时由服务器依据 `last_activity_time` 处理。

## 4. 上行协议与握手现状
当前实现并未定义自有二进制帧头或 TLV；协议要点如下：
- **Hello 消息**：
  ```json
  {
    "type": "hello",
    "audio_params": {"format": "opus"},
    "features": {"mcp": true, "kws": true}
  }
  ```
  服务器仅读取 `audio_params.format` 以设置 `conn.audio_format`（支持 `opus` 与 `pcm`），并根据 `features` 决定是否启用 MCP 等功能。
- **音频帧**：
  - WebSocket：直接发送单包 Opus（或 PCM）字节数据；
  - MQTT：在 `ConnectionHandler._process_mqtt_audio_message` 中解析 16 字节固定头（时间戳、长度），其余部分视为音频；该结构由 MQTT 网关处理，与本文关注的设备直连模式无关。
- **控制面**：除 `listen` 外，还有 `hello`、`abort`、`iot`、`mcp` 等消息类型，与 VAD 协同主要相关的是 `listen` 与唤醒文本。

## 5. 典型协同流程
1. 设备发送 `hello`，声明音频格式与能力；服务器初始化连接、加载 Silero VAD。
2. 设备开始推送音频帧：
   - 若端侧无 VAD：直接持续推流，服务器独立切分。
   - 若端侧有 VAD：
     - 在检测到语音时发送 `listen/start`（可附带自身缓存的首帧音频一起上传）；
     - 语音结束时发送 `listen/stop`，同时继续上传余下缓冲帧以覆盖结尾。
3. 服务器 Silero VAD 根据实时概率和端侧提示综合判定段落：
   - 判定语音持续时，将音频帧加入 `conn.asr_audio`；
   - 静音阈值满足时触发 ASR，对文本结果调用对话/意图处理。
4. 若用户在 TTS 播报时再次开口（端侧提示或 Silero 检测到语音），服务器调用 `handleAbortMessage` 打断播报，实现 barge-in。

## 6. 实践建议
- **端侧有 VAD**：
  - 维持一个 200–300ms 的本地环形缓冲，`listen/start` 时连同缓存一起发送，避免辅音被截断；
  - `listen/stop` 之后可继续推送数帧音频给服务器，配合 `conn.client_voice_stop` 的判定；
  - 若端侧还做了 ASR，可在 `detect` 消息中携带 `text`，服务器会直接进入对话流程。
- **端侧无 VAD**：
  - 直接推流即可，但要留意带宽；
  - 推荐使用 Opus 16k/20ms 帧，客户端无须关心分段，服务器会处理。
- **阈值调优**：
  - Silero VAD 阈值、静音时长等参数可以在配置中心调整（`config.yaml` 中的模型参数或 Manager-API 下发）。若端侧经常误报，可降低对端侧信号的信任度，将 `mode` 维持在 `auto`。

## 7. 后续扩展（规划）
目前服务器尚未实现文档化的统一帧头、TLV 扩展和 `flags.vad_*` 标记；若后续需要对不同芯片做细粒度对齐，可在现有实现基础上扩展：
- 在 WebSocket 二进制帧前附加统一元数据；
- 让服务器把端侧 `segment_id/chunk_id` 与自身队列对齐；
- 根据端侧的噪声、AEC 状态动态调整 Silero 阈值。

在现有版本中，上述扩展尚未落地，本文所述流程即为 `xiaozhi-server` 实际运行的双端 VAD 协同方式。
