# Memory 记忆系统

## 概述

Memory 系统是小智 ESP32 服务器的记忆管理模块，负责存储和检索对话历史中的关键信息，支持从简单的无记忆模式到复杂的专业记忆服务等多种实现方式。

## 系统架构

### 核心组件

1. **MemoryProviderBase** (`core/providers/memory/base.py`) - 抽象基类
2. **MemoryProvider** 具体实现类 - 位于各子目录
3. **工厂方法** (`core/utils/memory.py`) - 动态创建实例

### 支持的记忆类型

- **nomem** - 不使用记忆功能
- **mem_local_short** - 本地短期记忆
- **mem0ai** - Mem0AI 云端记忆服务

## 实现方式

### 1. nomem 方式

**文件位置**: `core/providers/memory/nomem/nomem.py`

**特点**:
- 最简单的实现方式
- 不保存任何记忆信息
- 适用于不需要记忆功能的场景

**代码实现**:
```python
class MemoryProvider(MemoryProviderBase):
    async def save_memory(self, msgs):
        logger.bind(tag=TAG).debug("nomem mode: No memory saving is performed.")
        return None

    async def query_memory(self, query: str) -> str:
        logger.bind(tag=TAG).debug("nomem mode: No memory query is performed.")
        return ""
```

**适用场景**:
- 隐私要求高的场景
- 每次对话都是独立的
- 追求简单高效的实现

### 2. mem_local_short 方式

**文件位置**: `core/providers/memory/mem_local_short/mem_local_short.py`

**特点**:
- 本地短期记忆实现
- 使用 LLM 进行智能记忆总结
- 支持记忆压缩和优化
- 数据保存在本地服务器

**核心功能**:
- **智能记忆总结**: 使用 LLM 总结对话关键信息
- **记忆结构化**: 采用 JSON 格式存储结构化记忆
- **动态优化**: 自动淘汰过期信息，保持记忆容量
- **本地存储**: 数据不上传外部服务器，保护隐私

**记忆结构示例**:
```json
{
  "时空档案": {
    "身份图谱": {
      "现用名": "张三丰",
      "特征标记": ["北京", "软件工程师", "养猫"]
    },
    "记忆立方": [...]
  }
}
```

### 3. mem0ai 方式

**文件位置**: `core/providers/memory/mem0ai/mem0ai.py`

**特点**:
- 专业的云端记忆服务
- 使用 Mem0AI 第三方服务
- 支持高级记忆功能

**核心功能**:
- **云端存储**: 使用 Mem0AI 云端服务
- **API 集成**: 通过 MemoryClient 与云端服务交互
- **用户隔离**: 支持多用户记忆隔离
- **错误处理**: 完善的错误处理和降级机制

## MemoryProvider 核心功能

### 1. 记忆保存接口
```python
async def save_memory(self, msgs):
    """保存新的记忆信息并返回记忆ID"""
```
- 接收对话消息列表
- 提取关键信息并保存
- 返回记忆ID用于后续引用

### 2. 记忆查询接口
```python
async def query_memory(self, query: str) -> str:
    """基于相似度查询特定角色的记忆"""
```
- 根据查询内容检索相关记忆
- 返回相关的历史记忆信息
- 支持语义相似度搜索

### 3. LLM 集成
```python
def set_llm(self, llm):
    self.llm = llm
```
- 绑定大语言模型进行智能记忆处理
- 支持不同的 LLM 提供商

### 4. 角色管理
```python
def init_memory(self, role_id, llm, **kwargs):
    self.role_id = role_id
    self.llm = llm
```
- 支持多角色记忆隔离
- 每个角色独立的记忆空间

## 实例创建流程

### 1. 配置文件设置
```yaml
selected_module:
  Memory: mem_local_short  # 或 nomem、mem0ai

Memory:
  mem_local_short:
    type: mem_local_short
    llm: ChatGLMLLM  # 独立的记忆处理模型
  mem0ai:
    type: mem0ai
    api_key: 你的mem0ai api key
  nomem:
    type: nomem
```

### 2. 工厂方法调用
```python
# 由 modules_initialize.py 调用
memory_provider = memory.create_instance(
    "mem_local_short", 
    config["Memory"]["mem_local_short"],
    summary_memory_config
)
```

### 3. 动态模块加载
```python
# memory.py 工厂方法
def create_instance(class_name, *args, **kwargs):
    # 检查文件是否存在
    if os.path.exists(os.path.join("core", "providers", "memory", class_name, f"{class_name}.py")):
        # 动态导入模块
        lib_name = f"core.providers.memory.{class_name}.{class_name}"
        if lib_name not in sys.modules:
            sys.modules[lib_name] = importlib.import_module(f"{lib_name}")
        # 创建实例
        return sys.modules[lib_name].MemoryProvider(*args, **kwargs)
```

## 为什么要创建 MemoryProvider 实例？

### 1. 抽象层设计
- 隐藏具体记忆实现的复杂性
- 提供统一的记忆管理接口
- 支持运行时切换不同的记忆策略

### 2. 状态管理
- 每个实例维护自己的配置和记忆空间
- 支持多个角色使用不同的记忆配置
- 可以绑定不同的 LLM 实例进行记忆处理

### 3. 扩展性
- 新增记忆服务只需实现基类接口
- 插件化架构，易于维护和扩展
- 支持第三方记忆服务集成

### 4. 数据安全
- 本地记忆服务保护用户隐私
- 云端记忆服务提供专业功能
- 灵活的记忆存储策略选择

## 使用示例

### 基本使用
```python
# 创建记忆服务实例
memory_provider = memory.create_instance("nomem", config)

# 初始化记忆空间
memory_provider.init_memory("user_123", llm_instance)

# 保存记忆
memory_id = await memory_provider.save_memory(dialogue_messages)

# 查询记忆
related_memories = await memory_provider.query_memory("用户的工作")
```

### 高级使用 (mem_local_short)
```python
# 创建本地短期记忆实例
memory_provider = memory.create_instance("mem_local_short", config)

# 设置 LLM 和初始化
memory_provider.set_llm(llm_instance)
memory_provider.init_memory("user_123", llm_instance)

# 保存对话记忆
messages = [
    {"role": "user", "content": "我叫张三，是北京的软件工程师"},
    {"role": "assistant", "content": "很高兴认识你，张三！"}
]
memory_id = await memory_provider.save_memory(messages)

# 查询相关记忆
memories = await memory_provider.query_memory("用户的工作地点")
# 返回结构化的记忆信息
```

### 云端记忆服务 (mem0ai)
```python
# 创建云端记忆实例
memory_provider = memory.create_instance("mem0ai", {
    "api_key": "your_mem0ai_key",
    "api_version": "v1.1"
})

# 保存记忆到云端
memory_id = await memory_provider.save_memory(dialogue_messages)

# 从云端查询记忆
memories = await memory_provider.query_memory("用户的爱好")
```

## 性能特性

### mem_local_short 方式的特性
- **本地处理**: 无需网络请求，响应快速
- **智能压缩**: 自动优化记忆容量
- **结构化存储**: 便于查询和分析
- **隐私保护**: 数据不离开本地服务器

### mem0ai 方式的特性
- **专业服务**: 使用专业的记忆管理服务
- **高级功能**: 支持更复杂的记忆操作
- **云端存储**: 支持跨设备记忆同步
- **API 集成**: 与第三方服务深度集成

## 扩展开发

### 新增记忆服务方式
1. 继承 `MemoryProviderBase` 类
2. 实现 `save_memory` 和 `query_memory` 方法
3. 在相应目录创建实现文件
4. 更新配置文件支持新的类型

### 新增记忆功能
1. 在 `save_memory` 中添加新的记忆处理逻辑
2. 在 `query_memory` 中添加新的检索算法
3. 支持更多记忆结构化格式

## 配置说明

### nomem 配置
```yaml
Memory:
  nomem:
    type: nomem
```

### mem_local_short 配置
```yaml
Memory:
  mem_local_short:
    type: mem_local_short
    llm: ChatGLMLLM  # 用于记忆处理的独立模型
```

### mem0ai 配置
```yaml
Memory:
  mem0ai:
    type: mem0ai
    api_key: 你的mem0ai api key
    api_version: v1.1
```

## 注意事项

1. **隐私保护**: 根据需求选择本地或云端记忆服务
2. **LLM 选择**: mem_local_short 方式建议使用轻量级模型
3. **容量管理**: 合理设置记忆容量限制，避免无限增长
4. **错误处理**: 确保在记忆服务失败时有合适的降级策略
5. **性能监控**: 关注记忆保存和查询的处理时间

---

*最后更新: 2025-08-19*