# 系统总体架构概览

> **说明：** 这是小智ESP32服务器的核心架构层次图，展示了主要的系统层级和组件关系。

## 系统分层架构

```mermaid
graph TB
    subgraph "外部接入层 (External Interface)"
        A[ESP32设备]
        B[管理后台API]
        C[MCP接入点]
        D[AI服务提供商]
    end
    
    subgraph "服务入口层 (Service Gateway)"
        E[WebSocketServer]
        F[SimpleHttpServer]
        G[主事件循环]
    end
    
    subgraph "连接处理层 (Connection Layer)"
        H[连接管理器]
        I[ConnectionHandler Pool]
        J[消息路由器]
    end
    
    subgraph "任务执行层 (Task Execution)"
        K[异步任务队列]
        L[线程池]
        M[AI服务调度器]
    end
    
    subgraph "核心服务层 (Core Services)"
        N[VAD语音检测]
        O[ASR语音识别]
        P[LLM大语言模型]
        Q[TTS语音合成]
        R[Memory记忆系统]
        S[Intent意图识别]
    end
    
    subgraph "插件扩展层 (Plugin System)"
        T[插件加载器]
        U[工具处理器]
        V[功能插件]
    end
    
    subgraph "数据存储层 (Data Layer)"
        W[音频缓冲队列]
        X[对话历史]
        Y[配置管理]
        Z[日志系统]
    end
    
    %% 连接关系
    A --> E
    B --> F
    C --> E
    
    E --> H
    F --> H
    G --> H
    
    H --> I
    I --> J
    
    J --> K
    J --> L
    K --> M
    L --> M
    
    M --> N
    M --> O
    M --> P
    M --> Q
    M --> R
    M --> S
    
    P --> D
    O --> D
    Q --> D
    
    %% Intent和Memory的调用关系
    P --> S
    S --> R
    R --> X
    S --> U
    
    T --> U
    U --> V
    
    I --> W
    I --> X
    H --> Y
    G --> Z
    
    %% 样式定义
    classDef external fill:#ffebee,stroke:#c62828,stroke-width:2px
    classDef gateway fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
    classDef connection fill:#f3e5f5,stroke:#6a1b9a,stroke-width:2px
    classDef execution fill:#e8f5e8,stroke:#2e7d32,stroke-width:2px
    classDef services fill:#fff3e0,stroke:#ef6c00,stroke-width:2px
    classDef plugin fill:#e0f2f1,stroke:#00695c,stroke-width:2px
    classDef data fill:#fce4ec,stroke:#ad1457,stroke-width:2px
    
    class A,B,C,D external
    class E,F,G gateway
    class H,I,J connection
    class K,L,M execution
    class N,O,P,Q,R,S services
    class T,U,V plugin
    class W,X,Y,Z data
```

## 架构层次说明

### 1. 外部接入层 (External Interface)
- **ESP32设备**：硬件客户端，通过WebSocket连接
- **管理后台API**：系统管理和监控接口
- **MCP接入点**：Model Context Protocol 集成点
- **AI服务提供商**：OpenAI、阿里云、豆包等第三方AI服务

### 2. 服务入口层 (Service Gateway)
- **WebSocketServer**：主要的WebSocket服务器
- **SimpleHttpServer**：HTTP API服务器
- **主事件循环**：统一的异步事件调度中心

### 3. 连接处理层 (Connection Layer)
- **连接管理器**：管理所有客户端连接的生命周期
- **ConnectionHandler Pool**：为每个连接创建独立的处理实例
- **消息路由器**：负责消息的分发和路由

### 4. 任务执行层 (Task Execution)
- **异步任务队列**：I/O密集型任务的异步处理
- **线程池**：CPU密集型任务的并行处理
- **AI服务调度器**：协调各AI服务的调用

### 5. 核心服务层 (Core Services)
- **VAD语音检测**：语音活动检测
- **ASR语音识别**：语音转文本
- **LLM大语言模型**：文本理解和生成
- **TTS语音合成**：文本转语音
- **Memory记忆系统**：对话历史管理
- **Intent意图识别**：用户意图理解

#### 核心服务调用关系
- **LLM → Intent**：LLM处理用户输入后调用Intent进行意图识别
- **Intent → Memory**：Intent根据识别结果更新或查询Memory中的对话历史
- **Intent → 工具处理器**：Intent识别到需要工具调用时触发插件系统
- **Memory → 对话历史**：Memory系统将对话数据持久化到数据存储层

### 6. 插件扩展层 (Plugin System)
- **插件加载器**：动态加载功能插件
- **工具处理器**：统一的工具调用处理
- **功能插件**：音乐播放、天气查询、IoT控制等

### 7. 数据存储层 (Data Layer)
- **音频缓冲队列**：音频数据的缓存和队列
- **对话历史**：会话记录和上下文
- **配置管理**：系统配置和参数
- **日志系统**：系统运行日志

## 核心设计原则

### 1. 分层解耦
每层职责清晰，通过标准接口通信，便于独立开发和测试。

### 2. 异步优先
采用异步编程模型，提高并发处理能力和资源利用率。

### 3. 插件化架构
核心功能可通过插件扩展，增强系统的可扩展性和灵活性。

### 4. 连接隔离
每个客户端连接拥有独立的处理上下文，确保多连接之间的隔离。

### 5. 服务化集成
AI服务通过统一接口集成，支持多种服务提供商的切换。

---

📋 **相关文档导航：**
- [02_连接管理架构](02_connection_management.md) - 连接处理层详细设计
- [03_AI服务集成架构](03_ai_services_integration.md) - AI服务层集成方式
- [04_数据流处理架构](04_data_flow.md) - 数据流向和处理流程
- [05_并发处理架构](05_concurrency_model.md) - 并发处理模型
- [06_生命周期管理架构](06_lifecycle_management.md) - 连接生命周期管理

*图表创建时间：2025-08-24*
