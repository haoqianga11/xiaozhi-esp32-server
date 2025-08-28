# 小智ESP32服务器性能优化指南

本文档详细介绍了小智ESP32服务器的性能优化配置和使用方法。

## 目录

1. [性能优化概述](#性能优化概述)
2. [配置文件说明](#配置文件说明)
3. [并发控制](#并发控制)
4. [数据库优化](#数据库优化)
5. [缓存优化](#缓存优化)
6. [监控和告警](#监控和告警)
7. [使用示例](#使用示例)
8. [常见问题](#常见问题)

## 性能优化概述

小智ESP32服务器经过性能优化后，可以支持更高的并发连接数和更快的响应速度。主要优化包括：

- **连接池优化**：增加数据库和Redis连接池大小
- **并发控制**：为AI服务提供智能并发控制和队列管理
- **缓存策略**：实现多层缓存机制，减少重复计算
- **超时控制**：优化各种超时设置，提高系统稳定性
- **监控告警**：实时监控系统状态，及时发现问题

## 配置文件说明

### 1. 性能配置文件

创建 `performance.yaml` 文件作为性能配置模板：

```yaml
# 服务器性能配置
server:
  max_connections: 500      # 最大并发连接数
  connection_timeout: 30    # 连接超时时间（秒）
  thread_pool:
    max_workers: 20         # 工作线程数
    queue_size: 100         # 任务队列大小

# AI服务并发控制
ai_providers:
  llm:
    max_concurrent: 10      # LLM最大并发数
    timeout: 30             # 请求超时时间
    enable_queue: true      # 启用请求队列
    queue_size: 50          # 队列大小
  tts:
    max_concurrent: 5       # TTS最大并发数
    timeout: 15
    enable_queue: true
    queue_size: 20
  asr:
    max_concurrent: 8       # ASR最大并发数
    timeout: 20
    enable_queue: true
    queue_size: 30

# 数据库连接池优化
database:
  mysql:
    max_active: 200         # 最大连接数
    min_idle: 20            # 最小空闲连接数
    initial_size: 20       # 初始连接数
    max_wait: 10000        # 最大等待时间（毫秒）
  redis:
    max_active: 50         # 最大连接数
    max_idle: 20           # 最大空闲连接数
    min_idle: 5            # 最小空闲连接数
```

### 2. 用户自定义配置

创建 `data/.performance.yaml` 文件来覆盖默认配置：

```yaml
# 用户自定义性能配置
server:
  max_connections: 300      # 降低最大连接数
  connection_timeout: 20    # 减少连接超时时间

ai_providers:
  llm:
    max_concurrent: 5       # 根据API限制调整
```

## 并发控制

### 1. 并发控制器

系统提供了智能并发控制器，用于管理AI服务的并发请求：

```python
from core.concurrency_controller import submit_ai_request, ServiceType

# 提交LLM请求
response = await submit_ai_request(
    ServiceType.LLM,
    "request_id",
    llm_function,
    args=(prompt,),
    kwargs={"temperature": 0.7}
)
```

### 2. 服务类型

支持的服务类型：
- `ServiceType.LLM`：大语言模型服务
- `ServiceType.TTS`：语音合成服务
- `ServiceType.ASR`：语音识别服务
- `ServiceType.VLLM`：视觉语言模型服务

### 3. 队列管理

当并发请求数超过限制时，请求会自动进入队列：

```python
# 检查队列状态
from core.concurrency_controller import get_concurrency_stats
stats = get_concurrency_stats()
print(f"LLM队列大小: {stats['queue_sizes']['llm']}")
```

## 数据库优化

### 1. MySQL连接池

已优化的MySQL连接池配置：

```yaml
spring:
  datasource:
    druid:
      initial-size: 20       # 初始连接数
      max-active: 200       # 最大连接数
      min-idle: 20          # 最小空闲连接数
      max-wait: 10000       # 最大等待时间
      validation-query: SELECT 1  # 连接验证查询
```

### 2. Redis连接池

已优化的Redis连接池配置：

```yaml
spring:
  data:
    redis:
      lettuce:
        pool:
          max-active: 50    # 最大连接数
          max-idle: 20      # 最大空闲连接数
          min-idle: 5       # 最小空闲连接数
          max-wait: 5000ms  # 最大等待时间
```

## 缓存优化

### 1. 多层缓存

系统实现了多层缓存机制：

- **Redis缓存**：分布式缓存，适合共享数据
- **内存缓存**：本地缓存，适合频繁访问的数据
- **AI响应缓存**：缓存AI模型响应，减少重复计算

### 2. 缓存配置

```yaml
cache:
  redis:
    enabled: true
    default_ttl: 3600      # 默认过期时间（秒）
    max_entries: 10000     # 最大缓存条目数
  memory:
    enabled: true
    max_size: 512          # 最大缓存大小（MB）
    default_ttl: 1800      # 默认过期时间（秒）
  ai_response:
    enabled: true
    similarity_threshold: 0.9  # 相似度阈值
    max_entries: 5000      # 最大缓存条目数
```

## 监控和告警

### 1. 性能监控

使用性能监控脚本：

```bash
# 实时监控
python performance_monitor.py

# 生成报告
python performance_monitor.py --report --duration 300 --output report.json

# 自定义监控间隔
python performance_monitor.py --interval 10 --duration 600
```

### 2. 监控指标

系统监控以下指标：

**系统指标：**
- CPU使用率
- 内存使用率
- 磁盘使用率
- 网络流量
- 系统负载

**应用指标：**
- 活跃连接数
- 请求响应时间
- 错误率
- 队列长度

### 3. 告警配置

```yaml
monitoring:
  alerts:
    cpu_threshold: 80               # CPU使用率告警阈值
    memory_threshold: 85            # 内存使用率告警阈值
    error_rate_threshold: 5         # 错误率告警阈值
    response_time_threshold: 5000    # 响应时间告警阈值
    active_connections_threshold: 400 # 活跃连接数告警阈值
```

## 使用示例

### 1. 基本使用

```python
# 1. 加载性能配置
from core.performance_config_loader import load_performance_config
config_loader = load_performance_config()

# 2. 应用性能配置
optimized_config = config_loader.get_optimized_config(base_config)

# 3. 初始化并发控制器
from core.concurrency_controller import init_concurrency_controller
init_concurrency_controller(optimized_config)

# 4. 启动性能监控
from performance_monitor import PerformanceMonitor
monitor = PerformanceMonitor(optimized_config)
monitor.start()
```

### 2. 自定义并发控制

```python
from core.concurrency_controller import ConcurrencyController, ServiceType, ServiceConfig

# 创建自定义并发控制器
controller = ConcurrencyController()

# 配置服务
controller.configure_service(
    ServiceType.LLM,
    ServiceConfig(
        max_concurrent=15,
        timeout=25,
        enable_queue=True,
        queue_size=100
    )
)

# 启动控制器
controller.start()
```

### 3. 性能监控集成

```python
from performance_monitor import PerformanceMonitor

# 创建监控器
monitor = PerformanceMonitor({
    'monitor_interval': 5,
    'thresholds': {
        'cpu_percent': 80,
        'memory_percent': 85
    }
})

# 添加告警回调
def alert_handler(alert):
    print(f"告警: {alert['message']}")

monitor.add_alert_callback(alert_handler)

# 启动监控
monitor.start()
```

## 常见问题

### 1. 配置不生效

**问题**：性能配置没有生效

**解决方案**：
1. 检查配置文件路径是否正确
2. 确认配置文件格式正确（YAML格式）
3. 查看日志中的配置加载信息
4. 使用 `data/.performance.yaml` 覆盖配置

### 2. 连接数过多

**问题**：数据库连接数过多导致连接超时

**解决方案**：
1. 增加数据库连接池大小
2. 启用连接池的连接验证
3. 检查是否有连接泄漏
4. 优化SQL查询性能

### 3. AI服务超时

**问题**：AI服务请求频繁超时

**解决方案**：
1. 调整AI服务的并发限制
2. 增加请求超时时间
3. 启用请求队列
4. 检查网络连接稳定性

### 4. 内存使用过高

**问题**：服务器内存使用率过高

**解决方案**：
1. 启用内存缓存限制
2. 调整线程池大小
3. 优化数据处理逻辑
4. 启用内存监控告警

### 5. 性能监控异常

**问题**：性能监控脚本无法正常运行

**解决方案**：
1. 确认已安装 `psutil` 库
2. 检查系统权限
3. 查看日志中的错误信息
4. 尝试降低监控频率

## 性能测试

### 1. 基准测试

```bash
# 运行性能测试
python performance_tester.py

# 生成性能报告
python performance_monitor.py --report --duration 300 --output performance_report.json
```

### 2. 压力测试

```bash
# 模拟多用户并发测试
python -c "
import asyncio
import concurrent.futures

async def test_connection():
    # 模拟连接测试
    pass

# 并发测试
with concurrent.futures.ThreadPoolExecutor(max_workers=50) as executor:
    futures = [executor.submit(test_connection) for _ in range(100)]
    concurrent.futures.wait(futures)
"
```

## 最佳实践

1. **逐步优化**：先小规模测试，再逐步扩大配置
2. **监控先行**：先启用监控，了解当前性能状况
3. **合理配置**：根据实际硬件资源和业务需求调整配置
4. **定期检查**：定期查看性能报告，及时发现问题
5. **文档记录**：记录配置变更和性能对比

## 技术支持

如果遇到性能问题，请：

1. 查看日志文件
2. 运行性能监控脚本
3. 收集系统性能数据
4. 提供详细的配置信息
5. 描述具体的业务场景

---

*本指南会根据项目发展持续更新，请关注最新版本。*