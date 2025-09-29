# AI服务集成架构

> **说明：** 结合当前代码实现，梳理 xiaozhi-server 的 VAD、ASR、LLM、TTS、Memory、Intent 等 AI 服务模块如何按配置装配、在运行期协同，以及如何与 MCP / 工具体系联动。

## 总体架构视图

```mermaid
graph TD
    cfg[配置文件\nconfig.yaml & data/.config.yaml] --> init[模块初始化\ncore/utils/modules_initialize.py]
    init --> ws[WebSocketServer\ncore/websocket_server.py]
    ws -->|新连接| conn[ConnectionHandler\ncore/connection.py]
    conn --> vad[VAD 提供者]
    conn --> asr[ASR 提供者]
    conn --> llm[LLM 提供者]
    conn --> tts[TTS 提供者]
    conn --> mem[Memory 提供者]
    conn --> intent[Intent 引擎]
    conn --> tools[UnifiedToolHandler\n工具/插件/MCP]
    asr --> voiceprint[VoiceprintProvider\n(可选)]
    tools --> mcp[MCP 客户端]
    tools --> iot[IoT / 插件工具]
```

- 配置集中声明在 `config.yaml` 与 `data/.config.yaml`，通过 `selected_module` 选择各类服务提供者 (`config.yaml:190`).
- 服务器启动时先调用 `initialize_modules` 预构建所需模块实例，再把这些实例注入每个连接处理器 (`core/utils/modules_initialize.py:9`, `core/websocket_server.py:14`).
- `ConnectionHandler` 在运行期维护音频、文本、工具调用等状态机，驱动各个提供者协同处理 (`core/connection.py:25`).
- MCP、IoT 等外部能力通过统一的工具路由层扩展，而非直接挂在 LLM 上 (`core/providers/tools/unified_tool_handler.py:18`).

## 组件装配流程

1. **配置读取**：`load_config()` 合并默认配置与 data 目录下的用户配置 (`config/settings.py:4`).
2. **模块初始化**：根据 `selected_module` 标记决定是否实例化 VAD/ASR/LLM/TTS/Memory/Intent；每类模块都有 `core/utils/<module>.py` 中的 `create_instance` 工厂函数 (`core/utils/vad.py:9`, `core/utils/asr.py:12`, `core/utils/llm.py:13`, `core/utils/tts.py:1`).
3. **共享实例复用**：WebSocketServer 在启动时创建共享实例并缓存，避免为每个连接重复冷启动重型模型 (`core/websocket_server.py:14`).
4. **连接期按需挂载**：`ConnectionHandler` 在 `handle_connection` 内按需克隆/绑定这些共享实例；TTS、ASR 会在连接生命周期中持有状态队列、线程池等资源 (`core/connection.py:71`).
5. **可选功能启用**：声纹识别、工具插件、MCP 接入等在连接建立时再按配置动态启用 (`core/utils/modules_initialize.py:132`, `core/providers/tools/unified_tool_handler.py:56`).

```python
# core/websocket_server.py:21 摘录
modules = initialize_modules(
    self.logger,
    self.config,
    "VAD" in self.config["selected_module"],
    "ASR" in self.config["selected_module"],
    "LLM" in self.config["selected_module"],
    False,
    "Memory" in self.config["selected_module"],
    "Intent" in self.config["selected_module"],
)
```

## 基类接口速览

| 模块 | 基类 | 关键方法 | 说明 | 参照 |
| --- | --- | --- | --- | --- |
| VAD | `VADProviderBase` | `is_vad(conn, data)` | 同步检测帧内是否存在语音活动，需管理连接内缓冲与阈值切换 | `core/providers/vad/base.py:5` |
| ASR | `ASRProviderBase` | `open_audio_channels`, `receive_audio`, `handle_voice_stop`, `speech_to_text(...)` | 负责音频累积、语音停止判定、可选声纹识别及最终文本生成 | `core/providers/asr/base.py:25` |
| LLM | `LLMProviderBase` | `response`, `response_no_stream`, `response_with_functions` | 标准输出为生成器流；非流式及函数调用在基类内做默认适配 | `core/providers/llm/base.py:7` |
| TTS | `TTSProviderBase` | `to_tts_stream`, `to_tts`, `text_to_speak` | 维护文本/音频队列，支持流式或文件式输出及失败重试 | `core/providers/tts/base.py:31` |
| Memory | `MemoryProviderBase` | `save_memory`, `query_memory`, `init_memory` | 统一记忆读写入口，可绑定独立 LLM | `core/providers/memory/base.py:7` |
| Intent | `Intent` 系列 | 依据实现暴露 `analyze` / `handle` 等接口 | 例如 `function_call`、`intent_llm` 构建在 LLM 之上做指令解析 | `core/providers/intent` |

## 提供商适配矩阵

> 具体可用项以 `config.yaml` 中模块列表为准，这里按目录/配置分类罗列常用组合。

### VAD
- `silero`：本地 Torch 模型，默认内置 (`core/providers/vad/silero.py:12`)。

### ASR
- **本地模型**：`fun_local` (SenseVoiceSmall)、`sherpa_onnx_local` (sense_voice / paraformer)、`vosk` (`core/providers/asr/fun_local.py`, `core/providers/asr/sherpa_onnx_local.py`, `core/providers/asr/vosk.py`)。
- **云/HTTP API**：阿里云 (`aliyun`, `aliyun_stream`)、火山引擎豆包 (`doubao`, `doubao_stream`)、腾讯 (`tencent`)、百度 (`baidu`)、讯飞 (`xunfei_stream`)、OpenAI/Groq (`openai`)、通义千问 Qwen3 (`qwen3_asr_flash`) 等 (`core/providers/asr/aliyun.py` 等)。

### LLM
- **OpenAI 兼容**：阿里云、火山引擎、ChatGLM、DeepSeek、Volces Gateway 等 (`core/providers/llm/openai`)。
- **专有实现**：`AliBL`（阿里应用版）、`coze`、`dify`、`fastgpt`、`gemini`、`homeassistant`、`ollama`、`xinference` 等 (`core/providers/llm/*`).

### TTS
- **本地**：`fishspeech`, `gpt_sovits_v2/v3` 等。
- **云服务**：微软 Edge、火山引擎（含双向流式）、阿里云、腾讯云、硅基流动、豆包、讯飞等 (`core/providers/tts/*.py`)。
- **自定义**：`custom`, `default` 适配器用于封装特殊流程。

### Memory 与 Intent
- Memory：`mem0ai`（云端）、`mem_local_short`（本地 SQLite）、`nomem`（禁用）。
- Intent：`function_call`（借助 LLM Function Call）、`intent_llm`（独立 LLM 推理）、`nointent`（禁用）。

### 工具 / MCP
- 工具统一由 `UnifiedToolHandler` 汇总，提供 **Server Plugins**、**Server MCP**、**Device IoT**、**Device MCP**、**MCP Endpoint** 五类执行器 (`core/providers/tools/unified_tool_handler.py:26`)。
- 设备侧 MCP 客户端负责工具列表同步与调用回执 (`core/providers/tools/device_mcp/mcp_handler.py:15`)。

## 运行期协作流程

1. **连接建立**：握手成功后创建 `ConnectionHandler`，完成认证、配置拉取与工具初始化 (`core/connection.py:115`)。
2. **音频接入**：客户端上行音频帧先经 VAD 判断是否有人声，并维护缓冲与静默窗口 (`core/providers/vad/silero.py:39`)。
3. **语音停止判定**：静默触发后，ASR 将累积音频转写文本，并可并行声纹识别 (`core/providers/asr/base.py:74`)。
4. **文本处理**：识别结果送入 `startToChat`，结合 Prompt、Memory、Intent、工具等决定回复策略 (`core/handle/receiveAudioHandle.py`, `core/providers/tools/unified_tool_handler.py:138`)。
5. **LLM 对话**：所选 LLM 以流式生成器输出，函数调用时通过 UnifiedToolHandler 调度执行器 (`core/providers/llm/base.py:15`)。
6. **语音合成**：TTS 负责将回复拆句、缓存、重试并推送给客户端，同时做产出统计 (`core/providers/tts/base.py:85`)。
7. **工具/插件**：若产生函数调用或设备侧请求，UnifiedToolHandler 将路由至插件、MCP 或 IoT 执行器，处理返回 (`core/providers/tools/device_mcp/mcp_handler.py:122`)。
8. **收尾控制**：`ConnectionHandler` 维护超时、退出指令与资源回收流程 (`core/connection.py:164`)。

## MCP 与外部能力联动

- 服务端可托管 MCP Server（如视觉解释接口），客户端也能作为 MCP Client 主动接入第三方工具；二者通过统一工具层协调 (`core/providers/tools/server_mcp`, `core/providers/tools/device_mcp`)。
- `config.yaml` 中的 `mcp_endpoint`、`manager-api` 配置控制外部工具启用，并在运行期验证格式 (`app.py:69`, `core/utils/util.py:489`)。

## 监控与容错现状

- 当前仓库没有独立的 `ServiceHealthMonitor`；健康检查多依赖 provider 内的重试、日志与异常处理。
- ASR/TTS 基类内置重试与性能日志，例如 TTS 的五次重试与音频队列 (`core/providers/tts/base.py:85`)，ASR 的并行声纹识别与耗时统计 (`core/providers/asr/base.py:74`)。
- 若需统一监控，可在 `core/utils/output_counter.py`、`core/handle/reportHandle.py` 等基础上扩展。

## 扩展指引

1. 在 `core/providers/<module>/` 下新增实现类，并确保 `core/utils/<module>.py` 的 `create_instance` 可以定位到对应文件。
2. 在 `config.yaml` 相应分支添加配置，同时更新 `selected_module` 选项。
3. 如果需要额外的工具或 MCP 集成，扩展 `core/providers/tools` 目录下的执行器，并在 `ToolManager` 中注册。

---

📋 **相关文档导航：**
- [01_系统总体架构](01_system_overview.md)
- [02_连接管理架构](02_connection_management.md)
- [04_数据流处理架构](04_data_flow.md)
- [VAD 设计与上行协议建议](VAD设计与上行协议建议byw.md)

*图表创建时间：2025-08-24（已根据当前代码调整内容）*
