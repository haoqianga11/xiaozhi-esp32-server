# xiaozhi-server 消息处理架构图

## 消息处理层整体关系

```mermaid
graph TB
    subgraph "客户端设备"
        WS[WebSocket连接]
        Audio[音频流]
        Text[文本消息]
    end

    subgraph "连接处理层"
        Auth[认证中间件]
        Config[配置获取]
        Route[Connection._route_message]
    end

    subgraph "文本消息处理器"
        TextRoute[handleTextMessage]
        TMP[TextMessageProcessor]
        Registry[HandlerRegistry]

        %% 文本消息处理器
        TMP -->|解析JSON| Registry
        Registry -->|根据type路由| HelloHandler["hello处理器"]
        Registry -->|根据type路由| AbortHandler["abort处理器"]
        Registry -->|根据type路由| IoTHandler["iot处理器"]
        Registry -->|根据type路由| ListenHandler["listen处理器"]
        Registry -->|根据type路由| McpHandler["mcp处理器"]
        Registry -->|根据type路由| ServerHandler["server处理器"]

        %% 处理器到对话的连接
        HelloHandler --> |设备能力协商| Chat
        IoTHandler --> |IoT控制| Chat
    end

    subgraph "音频处理队列"
        AudioQueue[asr_audio_queue]
        ASRThread[ASR处理线程]
        AudioBuffer[音频缓冲区]
        VADProcess[VAD检测]
        ASRProcess[语音识别]
        VoiceprintProcess[声纹识别]
        MQTTHandler[MQTT音频处理]
        IntentProcess[意图识别]

        %% 音频处理流程
        AudioQueue --> |队列消费| ASRThread
        ASRThread --> |音频处理| VADProcess
        VADProcess --> |音频累积| AudioBuffer
        AudioBuffer --> |语音停止| ASRProcess
        ASRProcess --> |并行处理| VoiceprintProcess
        VoiceprintProcess --> |声纹识别| ASRProcess
        ASRProcess --> |识别完成| startToChat[startToChat]
        MQTTHandler --> |时间戳处理| AudioQueue
    end

    subgraph "意图处理器"
        IntentProcess[handle_user_intent]
        DirectHandlers[直接处理器]
        LLMHandlers[LLM处理器]

        %% 意图处理流程
        startToChat --> |文本输入| IntentProcess
        IntentProcess --> |退出/唤醒词| DirectHandlers
        IntentProcess --> |LLM意图分析| DirectHandlers
        IntentProcess --> |未处理意图| LLMHandlers
        DirectHandlers --> |直接响应| Audio
        LLMHandlers --> |调用chat| Chat[Connection.chat]
    end

    subgraph "对话处理器"
        Chat[Connection.chat]
        LLM[LLM处理]
        FunctionCall[函数调用]
        TTS[TTS合成]
        Memory[记忆系统]

        %% 对话处理流程
        Chat --> |已判定模式| LLM
        Chat --> |函数调用| FunctionCall
        LLM --> |生成响应| TTS
        FunctionCall --> |执行工具| Chat
        Memory --> |记忆查询| Chat
        TTS --> |音频输出| Audio
    end

    subgraph "共享状态"
        ConnState[Connection状态]
        AudioBuffer[音频缓冲区]
        Dialogue[对话历史]
        Queue[消息队列]
        Plugins[插件系统]
    end

    %% 连接线
    WS --> Auth
    Auth --> Route
    Text --> TextRoute
    Audio --> Route

    %% 认证和配置连接
    Auth --> |设备认证| Config
    Config -.-> |配置更新| ConnState

    %% 音频处理连接
    Route --> |二进制音频| AudioQueue
    Route --> |MQTT音频| MQTTHandler

    %% 共享状态连接
    TextRoute -.-> |共享连接状态| ConnState
    ASRThread -.-> |音频缓冲区| AudioBuffer
    Chat -.-> |对话历史| Dialogue
    AudioQueue -.-> |队列系统| Queue
    FunctionCall -.-> |插件调用| Plugins

    %% 样式
    classDef processor fill:#e1f5fe,stroke:#0288d1,stroke-width:2px
    classDef handler fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
    classDef state fill:#e8f5e8,stroke:#388e3c,stroke-width:2px
    classDef component fill:#fff3e0,stroke:#ef6c00,stroke-width:2px

    class Route,TextRoute,AudioQueue,startToChat,Chat processor
    class HelloHandler,AbortHandler,IoTHandler,ListenHandler,McpHandler,ServerHandler handler
    class ConnState,AudioBuffer,Dialogue,Queue state
    class Auth,Config,Memory,Plugins component
    class IntentProcess,VoiceprintProcess,ASRProcess intent_component
```

## 详细消息处理流程

```mermaid
sequenceDiagram
    participant C as 客户端
    participant A as 认证中间件
    participant R as 消息路由
    participant T as 文本处理器
    participant Q as 音频队列
    participant P as 音频处理线程
    participant ST as startToChat
    participant D as 对话处理器
    participant L as LLM服务
    participant M as 记忆系统
    participant S as 共享状态

    %% 连接建立流程
    C->>A: 发送WebSocket连接请求
    A->>A: AuthMiddleware.authenticate()
    A->>R: 认证通过，建立连接
    R->>R: _initialize_private_config()
    R->>S: 获取差异化配置
    R->>R: executor.submit(_initialize_components)

    %% 文本消息处理流程
    C->>R: 发送JSON文本消息
    R->>R: _route_message() 判断为文本
    R->>T: handleTextMessage()
    T->>T: TextMessageProcessor.process_message()
    T->>T: 根据type选择Handler
    T->>S: 更新连接状态
    T->>D: 可能调用chat()

    %% 音频消息处理流程
    C->>R: 发送二进制音频数据
    R->>R: _route_message() 判断为音频
    alt MQTT网关音频
        R->>R: _process_mqtt_audio_message()
        R->>R: 解析16字节头部和时间戳
        R->>Q: 时间戳排序后放入队列
    else 普通音频
        R->>Q: 直接放入asr_audio_queue
    end
    Q->>P: ASR处理线程消费
    P->>P: VAD语音检测
    P->>S: 音频缓冲区累积
    Note over P,S: 语音停止时触发
    P->>P: 并行执行ASR和声纹识别
    P->>P: ASR语音识别
    P->>P: Voiceprint.identify_speaker()
    P->>P: 构建增强文本(包含说话人信息)
    P->>ST: startToChat(增强文本)

    %% 意图处理流程
    ST->>ST: handle_user_intent()
    ST->>ST: 解析JSON(如包含说话人信息)
    ST->>ST: 检查退出命令
    alt 意图已被处理(退出/唤醒词等)
        ST->>S: 直接生成响应
        S->>C: 音频输出
        Note over ST: 意图处理完成，不进入对话
    else 函数调用模式
        ST->>D: 直接调用chat()
        Note over ST: 跳过意图分析，直接对话
    else 需要LLM意图分析
        ST->>ST: analyze_intent_with_llm()
        alt 意图被识别并处理
            ST->>S: 直接生成响应
            S->>C: 音频输出
            Note over ST: 意图处理完成，不进入对话
        else 意图未被处理
            ST->>D: 调用chat()
            Note over ST: 进入常规对话流程
        end
    end

    %% 对话处理流程
    D->>D: 基于已判定模式处理
    alt 函数调用模式
        D->>M: 查询记忆
        M->>D: 返回记忆结果
        D->>L: LLM.response_with_functions()
        L->>D: 流式响应
        D->>D: 执行函数调用
        D->>D: 处理工具调用结果
    else 普通LLM模式
        D->>M: 查询记忆
        M->>D: 返回记忆结果
        D->>L: LLM.response()
        L->>D: 流式响应
    end
    D->>S: TTS队列
    S->>C: 音频输出

    %% 状态共享
    Note over C,S: 所有处理器共享连接状态
    Note over Q,P: 音频处理器使用共享队列
    Note over P: ASR阶段完成声纹识别
    Note over ST: 意图处理决定是否进入对话
    Note over D,M: 对话处理器维护对话历史和记忆
```

## 处理器职责分工表

| 处理器 | 主要职责 | 输入 | 输出 | 触发条件 |
|--------|----------|------|------|----------|
| **认证中间件** | 设备认证 | WebSocket Headers、设备ID | 认证结果 | 新建WebSocket连接 |
| **文本消息处理器** | 协议控制、设备管理 | JSON字符串 | 控制指令、状态更新 | 收到WebSocket文本消息 |
| **音频处理队列** | 音频数据管理、MQTT处理 | 二进制音频数据、MQTT音频包 | 排序后的音频数据 | 收到WebSocket音频消息 |
| **音频处理线程** | 语音检测、识别、声纹识别 | 音频缓冲区数据 | 增强文本(含说话人信息) | 音频队列被消费且语音停止 |
| **startToChat处理器** | 意图分析、请求分流 | 识别文本、JSON数据 | 直接响应或chat调用 | ASR识别完成或文本输入 |
| **对话处理器** | 智能对话、工具调用、记忆管理 | 文本查询、上下文 | 音频响应、函数结果 | 意图未被处理且需要LLM |
| **记忆系统** | 短期记忆、长期记忆查询 | 用户查询、角色ID | 记忆上下文 | 每次调用chat时 |
| **TTS系统** | 语音合成 | LLM响应文本 | 音频流 | LLM生成响应后 |

## 关键组件实现细节

### 1. 消息路由实现 (`Connection._route_message`)

消息路由是系统的核心，负责根据消息类型进行分发：

```python
async def _route_message(self, message):
    """消息路由"""
    if isinstance(message, str):
        # 文本消息路由到文本处理器
        await handleTextMessage(self, message)
    elif isinstance(message, bytes):
        # 音频消息处理
        if self.vad is None or self.asr is None:
            return

        # 处理来自MQTT网关的音频包
        if self.conn_from_mqtt_gateway and len(message) >= 16:
            handled = await self._process_mqtt_audio_message(message)
            if handled:
                return

        # 普通音频直接放入队列
        self.asr_audio_queue.put(message)
```

### 2. MQTT音频包处理

MQTT网关发送的音频包包含16字节头部，需要特殊处理：

```python
async def _process_mqtt_audio_message(self, message):
    """处理来自MQTT网关的音频消息，解析16字节头部"""
    try:
        # 提取头部信息
        timestamp = int.from_bytes(message[8:12], "big")
        audio_length = int.from_bytes(message[12:16], "big")

        # 提取音频数据
        if audio_length > 0 and len(message) >= 16 + audio_length:
            audio_data = message[16 : 16 + audio_length]
            # 基于时间戳进行排序处理
            self._process_websocket_audio(audio_data, timestamp)
            return True
    except Exception as e:
        self.logger.bind(tag=TAG).error(f"解析WebSocket音频包失败: {e}")
    return False
```

### 3. ASR和声纹识别并行处理 (`handle_voice_stop`)

ASR处理阶段会并行执行语音识别和声纹识别，然后生成增强文本：

```python
async def handle_voice_stop(self, conn, asr_audio_task):
    """处理语音停止事件"""
    # 并行执行ASR和声纹识别任务
    with ThreadPoolExecutor(max_workers=2) as executor:
        asr_future = executor.submit(run_asr)
        voiceprint_future = executor.submit(run_voiceprint)

        # 等待两个任务完成
        asr_result = asr_future.result()
        speaker_name = voiceprint_future.result()

    # 构建包含说话人信息的增强文本
    enhanced_text = self._build_enhanced_text(asr_result, speaker_name)

    # 调用startToChat进行意图处理
    await startToChat(conn, enhanced_text)
```

### 4. 意图处理流程 (`handle_user_intent`)

意图处理是系统的关键分流点，决定请求是否需要进入对话：

```python
async def handle_user_intent(conn, text):
    """意图处理主函数"""
    # 预处理输入文本，处理可能的JSON格式（包含说话人信息）
    try:
        if text.strip().startswith('{') and text.strip().endswith('}'):
            parsed_data = json.loads(text)
            if isinstance(parsed_data, dict) and "content" in parsed_data:
                text = parsed_data["content"]
                conn.current_speaker = parsed_data.get("speaker")
    except (json.JSONDecodeError, TypeError):
        pass

    # 检查是否有明确的退出命令
    if await check_direct_exit(conn, text):
        return True  # 意图已处理

    # 检查是否是唤醒词
    if await checkWakeupWords(conn, text):
        return True  # 意图已处理

    if conn.intent_type == "function_call":
        # 函数调用模式，跳过意图分析，直接进入对话
        return False

    # 使用LLM进行意图分析
    intent_result = await analyze_intent_with_llm(conn, text)
    if not intent_result:
        return False  # 意图未识别，进入对话

    # 处理识别到的意图
    return await process_intent_result(conn, intent_result, text)
```

### 5. 对话处理流程 (`Connection.chat`)

对话处理只在意图未被处理时执行，基于已判定的模式进行处理：

```python
def chat(self, query, depth=0):
    """对话处理主函数 - 只在意图未被处理时调用"""
    self.llm_finish_task = False

    # 为最顶层时新建会话ID和发送FIRST请求
    if depth == 0:
        self.sentence_id = str(uuid.uuid4().hex)
        self.dialogue.put(Message(role="user", content=query))
        # 发送TTS开始信号
        self.tts.tts_text_queue.put(TTSMessageDTO(
            sentence_id=self.sentence_id,
            sentence_type=SentenceType.FIRST,
            content_type=ContentType.ACTION,
        ))

    # 根据意图类型选择处理方式（已在调用前确定）
    functions = None
    if self.intent_type == "function_call" and hasattr(self, "func_handler"):
        functions = self.func_handler.get_functions()
    response_message = []

    try:
        # 每次chat都会查询记忆系统获取相关上下文
        memory_str = None
        if self.memory is not None:
            future = asyncio.run_coroutine_threadsafe(
                self.memory.query_memory(query), self.loop
            )
            memory_str = future.result()

        if self.intent_type == "function_call" and functions is not None:
            # 支持函数调用的LLM处理
            llm_responses = self.llm.response_with_functions(
                self.session_id,
                self.dialogue.get_llm_dialogue_with_memory(
                    memory_str, self.config.get("voiceprint", {})
                ),
                functions=functions,
            )
        else:
            # 普通LLM处理
            llm_responses = self.llm.response(
                self.session_id,
                self.dialogue.get_llm_dialogue_with_memory(
                    memory_str, self.config.get("voiceprint", {})
                ),
            )
    except Exception as e:
        self.logger.bind(tag=TAG).error(f"LLM 处理出错 {query}: {e}")
        return None

    # 处理流式响应
    tool_call_flag = False
    for response in llm_responses:
        if self.client_abort:
            break
        # 处理响应内容并填充 response_message
        # ... (流式响应处理逻辑)

    # 存储对话内容到历史记录（统一处理，适用于普通模式和函数调用模式）
    if len(response_message) > 0:
        text_buff = "".join(response_message)
        self.tts_MessageText = text_buff
        self.dialogue.put(Message(role="assistant", content=text_buff))
```

### 6. 记忆系统和对话历史维护机制

**记忆查询时机**：
- 每次调用`chat()`方法时都会执行记忆查询
- 为当前对话提供相关的历史上下文
- 支持短期记忆和长期记忆检索

**对话历史维护**：
- 每次chat结束时都会将助手响应存储到对话历史中
- 通过`self.dialogue.put(Message(role="assistant", content=text_buff))`实现
- 为后续对话提供完整的上下文支持

这种设计确保了每次对话都能获得完整的记忆支持，并且持续维护对话历史。

### 7. 认证和配置管理

每个连接都需要经过认证和配置初始化：

```python
async def handle_connection(self, ws):
    try:
        # 获取并验证headers
        self.headers = dict(ws.request.headers)

        # 进行认证
        await self.auth.authenticate(self.headers)

        # 认证通过后获取差异化配置
        self._initialize_private_config()
        # 异步初始化组件（包含记忆和意图识别）
        self.executor.submit(self._initialize_components)
```

## 系统特性说明

### 1. 多模态支持
- **文本消息**：JSON格式的控制指令和状态更新
- **音频消息**：支持实时语音识别和MQTT网关音频包
- **视觉消息**：通过HTTP接口提供图像识别服务

### 2. 智能处理能力
- **意图识别**：支持函数调用和LLM意图识别两种模式
- **记忆管理**：每次对话都查询记忆并维护对话历史，支持短期记忆和长期记忆
- **声纹识别**：基于用户语音特征的身份验证

### 3. 插件扩展性
- **工具调用**：支持IoT设备控制、MCP协议、插件系统
- **函数注册**：动态加载和注册功能插件
- **提供商抽象**：支持多种AI服务提供商

### 4. 容错和稳定性
- **连接管理**：自动重连和超时处理
- **错误恢复**：组件级错误隔离和恢复
- **资源管理**：线程池和内存使用优化
