# 连接管理与消息路由架构

> **说明：** 详细展示WebSocket连接管理和高并发消息路由机制的架构设计。

## WebSocket连接层架构（拆分）

> **说明：** 将大型架构图拆分为多个独立的子图，便于理解和展示各个组件的职责。

### 1. WebSocket服务器核心组件

```mermaid
graph TB
    subgraph "WebSocket服务器"
        A[WebSocketServer<br/>主服务器]
        B[连接监听器<br/>ConnectionListener]
        C[连接认证器<br/>ConnectionAuthenticator]
    end
    
    subgraph "连接流程"
        D[ESP32设备连接请求]
        E[连接建立]
        F[身份验证]
        G[创建ConnectionHandler]
    end
    
    D --> A
    A --> B
    B --> E
    E --> C
    C --> F
    F --> G
    
    classDef server fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
    classDef flow fill:#f1f8e9,stroke:#558b2f,stroke-width:2px
    
    class A,B,C server
    class D,E,F,G flow
```

### 2. 连接池管理结构

```mermaid
graph TB
    subgraph "连接池管理"
        A[ConnectionHandler 1<br/>session_id: abc123<br/>device_id: esp32_001]
        B[ConnectionHandler 2<br/>session_id: def456<br/>device_id: esp32_002]
        C[ConnectionHandler N<br/>session_id: xyz789<br/>device_id: esp32_N]
    end
    
    subgraph "连接注册表"
        D[ConnectionRegistry<br/>连接注册管理]
        E[Session映射表<br/>session_id → Handler]
        F[设备映射表<br/>device_id → session_id]
    end
    
    A --> D
    B --> D
    C --> D
    
    D --> E
    D --> F
    
    classDef connection fill:#f3e5f5,stroke:#6a1b9a,stroke-width:2px
    classDef registry fill:#e8f5e8,stroke:#2e7d32,stroke-width:2px
    
    class A,B,C connection
    class D,E,F registry
```

### 3. 连接独立资源配置

```mermaid
graph LR
    subgraph "连接1资源"
        A1[ConnectionHandler 1]
        B1[asr_audio_queue 1]
        C1[client_audio_buffer 1]
        D1[dialogue 1]
    end
    
    subgraph "连接2资源"
        A2[ConnectionHandler 2]
        B2[asr_audio_queue 2]
        C2[client_audio_buffer 2]
        D2[dialogue 2]
    end
    
    subgraph "连接N资源"
        AN[ConnectionHandler N]
        BN[asr_audio_queue N]
        CN[client_audio_buffer N]
        DN[dialogue N]
    end
    
    A1 --- B1
    A1 --- C1
    A1 --- D1
    
    A2 --- B2
    A2 --- C2
    A2 --- D2
    
    AN --- BN
    AN --- CN
    AN --- DN
    
    classDef connection fill:#f3e5f5,stroke:#6a1b9a,stroke-width:2px
    classDef resource fill:#fff3e0,stroke:#ef6c00,stroke-width:2px
    
    class A1,A2,AN connection
    class B1,C1,D1,B2,C2,D2,BN,CN,DN resource
```

### 4. 消息路由机制

系统采用两层消息路由架构，实现传输层和应用层的分离：

```mermaid
graph TB
    subgraph "消息输入 - 传输层分类"
        A[ESP32设备消息]
        B[字符串消息<br/>String<br/>JSON格式控制命令]
        C[字节消息<br/>Bytes<br/>音频数据流]
    end

    subgraph "第一层路由 - 传输层路由"
        D[_route_message<br/>connection.py:284]
    end

    subgraph "第二层路由 - 应用层路由"
        E[TextMessageProcessor<br/>textMessageProcessor.py]
    end

    subgraph "应用层消息处理器"
        F[Hello处理器<br/>连接问候]
        G[Abort处理器<br/>中断操作]
        H[Listen处理器<br/>语音控制]
        I[IoT处理器<br/>设备控制]
        J[MCP处理器<br/>MCP协议]
        K[Server处理器<br/>服务器管理]
        L[音频处理器<br/>VAD/ASR处理]
    end

    A --> B
    A --> C

    B --> D
    C --> D

    D --> E
    D --> L

    E --> F
    E --> G
    E --> H
    E --> I
    E --> J
    E --> K

    classDef input fill:#e1f5fe,stroke:#0277bd,stroke-width:2px
    classDef router1 fill:#e8f5e8,stroke:#2e7d32,stroke-width:2px
    classDef router2 fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
    classDef handler fill:#fff3e0,stroke:#ef6c00,stroke-width:2px

    class A,B,C input
    class D router1
    class E router2
    class F,G,H,I,J,K,L handler
```

**消息处理流程**：

1. **第一层路由**（`connection.py:_route_message`）：
   - 字符串消息 → 转发到 `handleTextMessage()`
   - 字节消息 → 转发到音频处理队列

2. **第二层路由**（`textMessageProcessor.py`）：
   - 解析JSON消息的 `type` 字段
   - 路由到对应的应用层处理器

3. **支持的消息类型**：
   - `hello`: 连接建立和身份验证
   - `abort`: 中断当前对话或操作
   - `listen`: 语音拾音状态控制（start/stop/detect）
   - `iot`: IoT设备控制命令
   - `mcp`: MCP协议通信
   - `server`: 服务器管理（重启、配置更新）

### 5. 异步任务管理层

```mermaid
graph TB
    subgraph "异步任务层"
        A[消息路由任务<br/>MessageRoutingTask]
        B[超时检查任务<br/>TimeoutCheckTask]
        C[配置更新任务<br/>ConfigUpdateTask]
        D[状态上报任务<br/>StatusReportTask]
    end
    
    subgraph "任务调度器"
        E[AsyncTaskScheduler<br/>异步任务调度器]
    end
    
    subgraph "监控目标"
        F[连接状态监控]
        G[系统配置监控]
        H[性能指标收集]
    end
    
    E --> A
    E --> B
    E --> C
    E --> D
    
    A --> F
    B --> F
    C --> G
    D --> H
    
    classDef async fill:#fce4ec,stroke:#ad1457,stroke-width:2px
    classDef scheduler fill:#e8eaf6,stroke:#3f51b5,stroke-width:2px
    classDef monitor fill:#f3e5f5,stroke:#6a1b9a,stroke-width:2px
    
    class A,B,C,D async
    class E scheduler
    class F,G,H monitor
```

### 6. Handler/Manager 协作视图

```mermaid
graph LR
    subgraph "接入层"
        WS[WebSocketServer<br/>连接监听]
        MOD[模块池初始化<br/>VAD/ASR/LLM]
    end

    subgraph "会话处理层"
        CH[ConnectionHandler<br/>单连接上下文]
        TMP[TextMessageProcessor]
        REG[TextMessageHandlerRegistry]
        H1[HelloTextMessageHandler]
        H2[IotTextMessageHandler]
        H3[ServerTextMessageHandler]
    end

    subgraph "会话私有资源"
        QUEUE[音频队列<br/>client_audio_buffer]
        DIALOGUE[Dialogue<br/>对话历史]
        VOICE[VoiceprintProvider<br/>声纹实例]
    end

    subgraph "管理与资源层"
        UTH[UnifiedToolHandler<br/>LLM函数路由]
        TM[ToolManager]
        EXEC[执行器集合<br/>Server/Device/MCP]
        PROMPT[PromptManager]
        CACHE[GlobalCacheManager]
    end

    WS --> CH
    MOD --> CH
    CH --> TMP
    TMP --> REG
    REG --> H1
    REG --> H2
    REG --> H3

    CH --> QUEUE
    CH --> DIALOGUE
    CH --> VOICE

    CH --> UTH
    UTH --> TM
    TM --> EXEC

    CH --> PROMPT
    PROMPT --> CACHE

    classDef entry fill:#e1f5fe,stroke:#0277bd,stroke-width:2px
    classDef handler fill:#fff3e0,stroke:#ef6c00,stroke-width:2px
    classDef manager fill:#e8f5e8,stroke:#2e7d32,stroke-width:2px
    classDef resource fill:#f3e5f5,stroke:#6a1b9a,stroke-width:2px

    class WS,MOD entry
    class CH,TMP,REG,H1,H2,H3 handler
    class UTH,TM,EXEC,PROMPT,CACHE manager
    class QUEUE,DIALOGUE,VOICE resource
```

> **理解要点：** 连接级 Handler 负责调度流程和维持会话状态，Manager 则集中承载共享资源（执行器、缓存、提示词等），两者通过清晰的接口解耦，便于扩展和故障隔离。

## WebSocket连接层架构（完整概览）

```mermaid
graph TB
    subgraph "WebSocket连接层完整架构"
        subgraph "接入层"
            WS[WebSocketServer<br/>连接监听与分发]
            AUTH[连接认证器<br/>ConnectionAuthenticator]
        end

        subgraph "连接管理层"
            REG[ConnectionRegistry<br/>连接注册管理]
            CH1[ConnectionHandler 1<br/>session_id: abc123]
            CH2[ConnectionHandler 2<br/>session_id: def456]
            CHN[ConnectionHandler N<br/>session_id: xyz789]
        end

        subgraph "消息路由层"
            ROUTER[MessageRouter<br/>消息路由中心]
            TEXT[TextMessageHandler<br/>文本消息处理]
            AUDIO[AudioMessageHandler<br/>音频消息处理]
            CTRL[ControlMessageHandler<br/>控制消息处理]
        end

        subgraph "资源管理层"
            UTH[UnifiedToolHandler<br/>统一工具处理器]
            TM[ToolManager<br/>工具管理器]
            PROMPT[PromptManager<br/>提示词管理]
            CACHE[GlobalCacheManager<br/>全局缓存]
        end

        subgraph "连接私有资源"
            Q1[asyncio.Queue 1<br/>音频队列]
            BUF1[bytearray 1<br/>音频缓冲区]
            DIALOGUE1[Dialogue 1<br/>对话历史]

            Q2[asyncio.Queue 2<br/>音频队列]
            BUF2[bytearray 2<br/>音频缓冲区]
            DIALOGUE2[Dialogue 2<br/>对话历史]

            QN[asyncio.Queue N<br/>音频队列]
            BUFN[bytearray N<br/>音频缓冲区]
            DIALOGUEN[Dialogue N<br/>对话历史]
        end

        subgraph "AI服务层"
            ASR[ASR Provider<br/>语音识别]
            LLM[LLM Provider<br/>大语言模型]
            TTS[TTS Provider<br/>语音合成]
            VAD[VAD Provider<br/>语音活动检测]
        end
    end

    WS --> AUTH
    AUTH --> REG
    REG --> CH1
    REG --> CH2
    REG --> CHN

    CH1 --> ROUTER
    CH2 --> ROUTER
    CHN --> ROUTER

    ROUTER --> TEXT
    ROUTER --> AUDIO
    ROUTER --> CTRL

    CH1 --> Q1
    CH1 --> BUF1
    CH1 --> DIALOGUE1

    CH2 --> Q2
    CH2 --> BUF2
    CH2 --> DIALOGUE2

    CHN --> QN
    CHN --> BUFN
    CHN --> DIALOGUEN

    TEXT --> UTH
    AUDIO --> UTH
    CTRL --> UTH

    UTH --> TM
    UTH --> PROMPT
    PROMPT --> CACHE

    TM --> ASR
    TM --> LLM
    TM --> TTS
    TM --> VAD

    classDef entry fill:#e1f5fe,stroke:#0277bd,stroke-width:2px
    classDef connection fill:#f3e5f5,stroke:#6a1b9a,stroke-width:2px
    classDef router fill:#e8f5e8,stroke:#2e7d32,stroke-width:2px
    classDef manager fill:#fff3e0,stroke:#ef6c00,stroke-width:2px
    classDef resource fill:#f3e5f5,stroke:#6a1b9a,stroke-width:2px
    classDef service fill:#e8f5e8,stroke:#2e7d32,stroke-width:2px

    class WS,AUTH entry
    class REG,ROUTER router
    class CH1,CH2,CHN connection
    class UTH,TM,PROMPT,CACHE manager
    class Q1,BUF1,DIALOGUE1,Q2,BUF2,DIALOGUE2,QN,BUFN,DIALOGUEN resource
    class TEXT,AUDIO,CTRL,ASR,LLM,TTS,VAD service
```

> **完整架构说明：** 该架构图展示了WebSocket连接层的完整体系结构，从连接接入、消息路由到资源管理和AI服务集成的全流程。每个ConnectionHandler维护独立的异步资源队列，确保高并发场景下的性能和稳定性。

## 高并发连接架构设计

### 1. 连接状态隔离机制

每个 `ConnectionHandler` 实例维护完全独立的状态空间：

```python
class ConnectionHandler:
    def __init__(self):
        # 连接唯一标识
        self.session_id = str(uuid.uuid4())  # 唯一会话ID
        self.device_id = None               # 设备ID
        self.client_ip = None               # 客户端IP

        # 连接级别的队列和缓冲区
        self.asr_audio_queue = asyncio.Queue()    # 每个连接独立的异步音频队列
        self.client_audio_buffer = bytearray()    # 每个连接独立的音频缓冲区
        self.dialogue = Dialogue()               # 每个连接独立的对话历史

        # 连接状态管理
        self.websocket = None               # WebSocket连接实例
        self.stop_event = asyncio.Event()   # 停止信号
        self.last_activity = time.time()    # 最后活动时间
```

### 2. 消息路由机制

```mermaid
sequenceDiagram
    participant ESP32 as ESP32设备
    participant WS as WebSocketServer
    participant CH as ConnectionHandler
    participant Router as 消息路由器
    participant Handler as 消息处理器

    ESP32->>WS: 建立WebSocket连接
    WS->>CH: 创建ConnectionHandler实例
    CH->>CH: 初始化独立状态空间
    
    ESP32->>CH: 发送消息(文本/音频/控制)
    CH->>Router: 路由消息(传递conn实例)
    
    alt 文本消息
        Router->>Handler: handleTextMessage(conn, message)
        Handler->>Handler: 操作conn.dialogue
    else 音频消息  
        Router->>Handler: handleAudioMessage(conn, audio)
        Handler->>Handler: 存入conn.asr_audio_queue
    else 控制消息
        Router->>Handler: handleControlMessage(conn, control)
        Handler->>Handler: 更新conn状态
    end
    
    Handler->>CH: 返回处理结果
    CH->>ESP32: 发送响应
```

### 3. 连接池管理架构

```mermaid
graph TB
    subgraph "连接注册表"
        A[ConnectionRegistry]
        B[session_id -> ConnectionHandler映射]
        C[device_id -> session_id映射]
        D[连接统计信息]
    end
    
    subgraph "连接生命周期管理"
        E[连接创建]
        F[连接认证]
        G[连接活跃监控]
        H[连接清理]
    end
    
    subgraph "资源监控"
        I[内存使用监控]
        J[队列深度监控]
        K[连接数量监控]
        L[性能指标收集]
    end
    
    subgraph "异常处理"
        M[僵尸连接检测]
        N[资源泄露防护]
        O[异常连接清理]
        P[故障隔离]
    end
    
    A --> B
    A --> C
    A --> D
    
    E --> F
    F --> G
    G --> H
    
    G --> I
    G --> J
    G --> K
    G --> L
    
    I --> M
    J --> M
    K --> M
    M --> N
    N --> O
    O --> P
    
    classDef registry fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
    classDef lifecycle fill:#f3e5f5,stroke:#6a1b9a,stroke-width:2px
    classDef monitoring fill:#e8f5e8,stroke:#2e7d32,stroke-width:2px
    classDef exception fill:#ffebee,stroke:#c62828,stroke-width:2px
    
    class A,B,C,D registry
    class E,F,G,H lifecycle
    class I,J,K,L monitoring
    class M,N,O,P exception
```

## 消息路由核心机制

### 1. 消息类型识别与分发

```python
async def _route_message(self, message):
    """消息路由核心逻辑"""
    try:
        if isinstance(message, str):
            # 文本消息处理
            await self._handle_text_message(message)
        elif isinstance(message, bytes):
            # 音频消息处理
            await self._handle_audio_message(message)
        elif isinstance(message, dict):
            # JSON控制消息处理  
            await self._handle_control_message(message)
        else:
            logger.warning(f"未知消息类型: {type(message)}")
            
    except Exception as e:
        logger.error(f"消息路由异常: {e}")
        await self._handle_routing_error(e)
```

### 2. 连接上下文传递

```python
async def handleTextMessage(conn, message):
    """文本消息处理 - 连接上下文传递"""
    # 获取连接特定的状态
    device_id = conn.device_id
    session_id = conn.session_id
    websocket = conn.websocket
    
    # 操作连接独立的对话历史
    conn.dialogue.add_message("user", message)
    
    # 基于连接状态进行处理
    response = await process_user_input(conn, message)
    
    # 通过连接实例发送响应
    await conn.send_message(response)

async def handleAudioMessage(conn, audio_data):
    """音频消息处理 - 连接隔离"""
    # 存入连接专属的音频队列（异步操作）
    await conn.asr_audio_queue.put(audio_data)

    # 基于连接状态进行VAD检测
    have_voice = conn.vad.is_vad(conn, audio_data)

    if have_voice:
        # 添加到连接专属的音频缓冲区
        conn.client_audio_buffer.extend(audio_data)
```

## 并发处理优势

### 1. 状态隔离优势
- **完全独立**：每个连接的状态完全隔离，无共享状态
- **故障隔离**：单个连接出错不会影响其他连接
- **配置独立**：每个连接可以有不同的AI服务配置

### 2. 资源独立优势
- **队列独立**：每个连接有独立的消息队列
- **缓冲独立**：音频缓冲区按连接隔离
- **历史独立**：对话历史按会话管理

### 3. 扩展性优势
- **线性扩展**：连接数增加不影响现有连接性能
- **高并发支持**：可轻松支持数千个并发连接
- **动态负载**：根据连接负载动态调整资源

## 性能优化设计

### 1. 异步I/O优化
```python
class OptimizedConnectionHandler:
    def __init__(self):
        # 使用异步队列替代同步队列
        self.message_queue = asyncio.Queue(maxsize=1000)
        self.response_buffer = collections.deque(maxlen=100)
        
    async def process_messages_batch(self):
        """批量处理消息以提升性能"""
        batch = []
        try:
            # 批量收集消息
            while len(batch) < 10:
                message = await asyncio.wait_for(
                    self.message_queue.get(), timeout=0.1
                )
                batch.append(message)
        except asyncio.TimeoutError:
            pass
            
        if batch:
            await self.process_message_batch(batch)
```

### 2. 内存管理优化
```python
class MemoryOptimizedHandler:
    def __init__(self):
        self.max_buffer_size = 5 * 1024 * 1024  # 5MB
        self.max_queue_size = 1000
        self.max_dialogue_length = 50
        
    async def cleanup_resources(self):
        """定期资源清理"""
        # 清理超大音频缓冲
        if len(self.client_audio_buffer) > self.max_buffer_size:
            self.client_audio_buffer = self.client_audio_buffer[-self.max_buffer_size:]
            
        # 清理队列积压 - 丢弃最旧消息，避免替换队列实例
        while self.asr_audio_queue.qsize() > self.max_queue_size:
            try:
                dropped_msg = self.asr_audio_queue.get_nowait()
            except asyncio.QueueEmpty:
                break
            else:
                # 标记被丢弃的任务完成，防止 queue.join() 失真
                self.asr_audio_queue.task_done()
                # 也可以在此处记录 dropped_msg 方便排查
            
        # 清理对话历史
        if len(self.dialogue.dialogue) > self.max_dialogue_length:
            self.dialogue.dialogue = self.dialogue.dialogue[-self.max_dialogue_length:]
```

## 监控与告警

### 1. 关键指标监控
- **连接数量**：当前活跃连接数、新增连接率
- **消息吞吐**：每秒处理消息数、消息积压情况  
- **资源使用**：每连接内存占用、队列深度
- **响应延迟**：消息处理延迟分布、P95/P99延迟

### 2. 异常检测与处理
- **僵尸连接检测**：超时无响应连接自动清理
- **资源泄露防护**：内存/队列溢出保护机制
- **连接异常隔离**：异常连接不影响其他连接
- **自动恢复机制**：连接断开后的自动重连支持

---

📋 **相关文档导航：**
- [01_系统总体架构](01_system_overview.md) - 系统整体架构概览
- [03_AI服务集成架构](03_ai_services_integration.md) - AI服务层集成方式
- [06_生命周期管理架构](06_lifecycle_management.md) - 连接生命周期详细管理

*图表创建时间：2025-08-24*
