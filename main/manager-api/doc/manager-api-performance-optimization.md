# manager-api 性能优化方案

## 0. 性能分析
### 潜在瓶颈

- 设备心跳的异步刷新依赖一个极小线程池（核心2/最大4、队列1000），突发连接会迅速回落到请求线程并阻塞 Tomcat；参考 main/manager-api/src/main/java/xiaozhi/common/config/AsyncConfig.java:18-38 与 main/manager-api/src/main/java/xiaozhi/modules/device/service/impl/DeviceServiceImpl.java:206-215。
- 热点设备查询（getUserDevices、deleteByAgentId、selectCountByUserId 等）缺少 user_id/agent_id 索引，目前表结构仅有 mac_address 索引；当 ai_device 行数上万时会导致 MySQL 全表扫描；见 main/manager-api/src/main/java/xiaozhi/modules/device/service/impl/DeviceServiceImpl.java:223-267 与建表脚本 main/manager-api/src/main/resources/db/changelog/202503141346.sql:110-127。
- OTA 检查频繁调用 sysParamsService.getValue(..., false) 绕过 Redis 缓存（如 server.mqtt_gateway、server.mqtt_signature_key），使每次握手都额外打 MySQL；见 main/manager-api/src/main/java/xiaozhi/modules/device/service/impl/DeviceServiceImpl.java:191-203、:359-369 以及 main/manager-api/src/main/java/xiaozhi/modules/device/controller/DeviceController.java:153-169。
- 设备状态转发接口每次都遍历全部绑定设备拼装大 JSON 并推送到 MQTT 管理端，绑定量大时调用成本呈线性增长；位于 main/manager-api/src/main/java/xiaozhi/modules/device/controller/DeviceController.java:95-147。
- Tomcat 允许 1000 并发线程，而数据源只配置 100 个活跃连接，遇到 1 万设备短时重连会迅速耗尽连接池并阻塞请求；配置见 main/manager-api/src/main/resources/application.yml:2-8 与 main/manager-api/src/main/resources/application-dev.yml:10-38。
- 心跳无条件 updateById 回写 last_connected_at（甚至固件信息），高频上报时会造成大量写入和 binlog 膨胀；参考 main/manager-api/src/main/java/xiaozhi/modules/device/service/impl/DeviceServiceImpl.java:66-82。

### 建议

- 调整/替换异步线程池，使其规模匹配预期 QPS，并加监控观察队列长度。
- 为 ai_device 增加 (user_id, agent_id)、(agent_id) 等复合索引后重新执行 explain 验证。
- 将热点参数改为走缓存（或启动时预加载），避免每次握手都查库。
- 在设备状态转发前增加分页/限额或缓存，仅发送增量信息。
- 重新评估 Tomcat 和数据库连接池的容量，并用模拟 1 万设备的压测验证调整结果。

完成这些优化后，manager-api 在万级设备并发场景下会更稳健。

## 1. 目标
- 支撑 10k+ 设备在线/重连场景，确保核心接口 P99 < 1.5s、错误率 < 0.5%
- 减少突发场景下的数据库、Redis、线程池饱和风险
- 建立持续可验证的容量规划与监控体系

## 2. 现状关键瓶颈
- OTA 心跳异步写库线程池 (AsyncThread-*) 核心 2 / 最大 4，队列 1000，溢出后回落到请求线程，Tomcat 被阻塞
- ai_device 表仅有 mac_address 索引，大量根据 user_id/agent_id 的查询会触发全表扫描
- 热点系统参数调用 sysParamsService.getValue(..., false)，绕过 Redis，每次都查库
- /device/bind/{agentId} 聚合所有设备后转发到 MQTT 管理端，O(n) 序列化与请求体在高绑定量下耗时
- Tomcat (maxThreads=1000) 与 Druid (maxActive=100) 不匹配，引发连接池耗尽、线程排队
- 心跳每次 updateById 写 last_connected_at，高频写放大 binlog 与锁竞争

## 3. 优化设计

### 3.1 异步任务与写入削峰
- 调整线程池：corePoolSize=16、maxPoolSize=64、queueCapacity=20000，拒绝策略改为自定义丢队列指标并阻塞等待（或使用 CallerRunsPolicy + 重试限制）
- 采用批量写：心跳数据写入内存队列，使用 @Scheduled(fixedRate=200ms) 批量 UPDATE，减少单条写
- 监控：暴露 ThreadPoolTaskExecutor 指标到 Micrometer，关键指标 queueSize、activeCount，设置告警

### 3.2 数据库结构与查询优化
- 索引调整：
  ```sql
  ALTER TABLE ai_device ADD INDEX idx_ai_device_user_agent (user_id, agent_id);
  ALTER TABLE ai_device ADD INDEX idx_ai_device_agent (agent_id);
  ALTER TABLE ai_device ADD INDEX idx_ai_device_user (user_id);
  ```
- 查询优化：selectCountByUserId、getUserDevices 改为使用只读事务/缓存（例如 Redis 缓存 agentId 的设备数、列表，心跳写入时增量更新/失效处理）
- 对 OTA getLatestLastConnectionTime 增加异步缓存预热，避免频繁回库

### 3.3 Redis 连接池与缓存策略
- 连接池配置分析：当前 `application-dev.yml` 中 `spring.data.redis.lettuce.pool.max-active=8`、`max-idle=8`，远低于 10k 设备产生的心跳/指令请求量。建议提升至 `max-active=200` 以上，并根据压测调优 `max-wait`（1-3s）和 `min-idle`，确保突发连接不会被拒。必要时启用连接复用（Lettuce 自带）与命令管道化，降低 RT。
- 资源隔离与监控：将 Redis 指标（连接数、命令 QPS、慢查询、阻塞等待）纳入 Grafana 面板，设置连接耗尽/等待超时告警。对管理端、设备端采用不同的前缀或逻辑库，避免热点互相干扰。
- 热点缓存策略：
  - 使用 `sysParamsService.getValue(code, true)` 获取热点参数，必要时在项目启动时预加载 `server.mqtt_gateway`、`server.mqtt_signature_key` 等配置
  - 为设备注册验证码等高频 key 设置 TTL（如 10 分钟），防止 Redis 常驻键过多
  - 心跳写库前判断 appVersion 是否变化，未变则跳过写入避免无效更新

### 3.4 设备状态转发与接口设计
- /device/bind/{agentId} 接口增加分页参数或限制返回数量；前端仅在必要时请求全量
- 引入“设备状态聚合服务”：定时从 MQTT 管理端同步并落地缓存，API 直接读取缓存
- 对外请求改为异步（WebFlux 或消息队列），将耗时 I/O 与用户请求解耦

### 3.5 OTA 接口吞吐优化
- 前端校验 MAC 合法性后再上报，减少无效请求
- 复用生成的 MQTT token：缓存每日 token，或在项目内全局懒加载生成
- `buildMqttConfig` 中的 ObjectMapper 替换为单例，避免频繁实例化

### 3.6 WebSocket 连接管理
- 目标：支撑万级设备与 Web 管理端的长连接订阅，防止单节点内存或文件描述符耗尽。
- 连接接入：基于 Spring WebSocket/Netty，开启心跳（ping/pong）与空闲连接清理，设置 `setTaskScheduler` 和 `setSendTimeLimit` 保证消息有界。
- 资源限制：调高 `server.tomcat.max-connections` 或切换到 Undertow，结合 `ulimit -n` 扩大文件描述符数；使用 Redis/内存记录连接数，超过阈值时引导到其他节点。
- 状态管理：使用共享存储（Redis pub/sub 或消息总线）广播设备状态，确保水平扩展后不同节点能同步消息；对频繁推送的主题实施节流和合并。

### 3.7 JVM 内存与 GC 调优
- GC 策略：推荐使用 G1GC，设置 `-XX:+UseG1GC -XX:MaxGCPauseMillis=200`，并根据压测调优 `InitiatingHeapOccupancyPercent` 以适配设备心跳峰值；为大对象分配预留 `-XX:G1HeapRegionSize=4m`。
- 内存分配：把堆大小固定在 4-6GB（依据主机规格），并为 Netty/WebSocket 预留足够的直接内存，对日志/缓存的对象池化，避免频繁创建大对象。
- 监控与诊断：开启 `-XX:+PrintGCDetails -XX:+PrintGCDateStamps`，结合 Prometheus JMX Exporter 采集 `jvm_memory_used_bytes`、`jvm_gc_pause_seconds`；在压测和故障时使用 MAT、Async-Profiler 定位内存泄漏。

### 3.8 基础设施调优
- Tomcat：根据压测调低 `maxThreads` (~400)，提高 `acceptCount`，防止线程上下文切换过多
- Druid：提升 `maxActive` 至 300，`maxWait` 60000ms，并开启连接泄露检测
- MySQL：确保 `max_connections` >= Web/Job 合计峰值 + 安全余量；开启慢查询日志，定期 review

### 3.9 水平扩展与分片策略
- 应用层：manager-api 尽量保持无状态，使用 JWT 或 Redis Session 存储用户态信息，结合 Nginx/SLB 做七层负载均衡，实现多实例部署。
- 数据库层：按照用户/agent 维度做逻辑分片或读写分离；热点设备数据可迁移到独立库或使用 MySQL InnoDB Cluster，降低单库压力。
- 消息链路：为 MQTT、WebSocket 推送拆分独立节点，通过 Redis Stream/Kafka 做消息总线，避免单点瓶颈。
- 自动扩缩容：结合 K8s HPA/Cluster Autoscaler，根据 CPU、QPS、连接数指标进行弹性伸缩；提供扩容 Runbook。

### 3.10 测试与验证
- 构建模拟 10k 设备心跳/重连的压测脚本，覆盖 OTA、绑定、命令下发等关键链路
- 观测指标：Tomcat 线程、JDBC 连接、Redis QPS、Redis 等待时间、WebSocket 在线数、JVM GC、MySQL TPS、99 分位耗时
- 压测前后对比：写入放量能力、热点参数访问量、线程池/连接池饱和情况

### 3.11 监控与运维
- 统一接入 Prometheus/Grafana 或 SkyWalking，采集 JVM、线程池、Redis、MySQL、WebSocket 指标
- 设置告警阈值：线程池 `queueSize>80%`、数据库或 Redis 连接使用率 `>85%`、心跳接口 `P99>1.5s`、GC 暂停 `>500ms`
- 定期回顾设备增长量，更新容量规划预案；制定故障演练（Redis 拒绝连接、WebSocket 断链、数据库故障切换等）

## 4. 实施路线
1. **准备阶段（~1 周）**：完善压测脚本，部署监控，确认数据库与 Redis 配置变更窗口
2. **结构优化（~2 周）**：调整线程池 + Redis/数据库参数、添加索引、分页接口，完成回归测试
3. **专项压测（~1 周）**：模拟 10k 设备，记录指标；根据结果继续调参
4. **上线与观测**：分批上线，监控 72 小时，若无异常再开放更高设备量

## 5. 预期收益
- 设备心跳吞吐提升 5~10 倍，写库峰值平滑
- MySQL/Redis 查询耗时与连接竞争大幅下降，CPU 使用率下降
- 系统参数查询基本命中缓存，后台 QPS 降至原来的 10%
- WebSocket、MQTT、HTTP 等多协议链路在 10k 设备场景下依旧保持 P99 <1.5s
