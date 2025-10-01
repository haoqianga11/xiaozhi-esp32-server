# 短期方案
- core/connection.py:93 当前为每个连接创建 ThreadPoolExecutor(max_workers=5)，调整为在模块级构建单例线程池（如 shared_executor = ThreadPoolExecutor(max_workers=16)）并注入到 ConnectionHandler；实例化时引用共享池，关闭连接时不再 shutdown 全局池，仅在进程退出时统一回收。
- 在 core/providers/tts/base.py:232 和 core/providers/asr/base.py:21 为 threading.Thread 初始化处增加连接线程命名与活动计数，结合 threading.active_count() 与 psutil.Process().num_threads() 写到自定义监控钩子，便于 Prometheus/自建监控采集。
- core/websocket_server.py（接入点）增加连接上限控制，采用 asyncio.Semaphore 包装 websockets.serve 的 handler，超过阈值立即返回 503 或自定义错误，阈值初期建议按照可承载线程数（如 200 设备）设定。
- 在 ConnectionHandler.close()（core/connection.py:1008）确保 stop_event 置位并向队列 put(None)，保证现有线程可以在超时内退出；同时增加 join(timeout=5)（捕获异常以防阻塞）确认守护线程平滑结束。
- 为 core/providers/tts/base.py 和 core/providers/asr/base.py 新增配置开关，允许在配置文件中逐步切换到共享线程池或异步实现；短期目标是使各连接线程消耗降为：1 个常驻守护线程 + 若干共享池任务。

发布前进行压力测试：编写脚本批量发起 WebSocket 连接，观察监控指标（线程数、RSS、上下文切换），记录基线并验证线程共享策略是否生效。

# 长期方案

- 将 TTS/ASR 队列迁移至 asyncio.Queue，消费逻辑改成协程：在 ConnectionHandler 初始化时 self.tts_queue = asyncio.Queue()、self.asr_queue = asyncio.Queue()，并在事件循环里通过 asyncio.create_task(self._tts_consumer()) 等方式启动；原 threading.Thread 被 asyncio 任务取代。
- 对于 CPU/阻塞型操作（TTS 合成、文件 IO、Opus 编解码），在协程中 loop.run_in_executor(shared_executor, blocking_fn, ...) 调用全局线程池，确保 Python 侧仍可并行；封装 tts_provider.synthesize_async() / asr_provider.recognize_async() 以统一入口。
- 在 core/providers/tts/base.py 设计新的异步接口：async def process_message(self, message: TTSMessageDTO)，内部根据消息类型串联文本拼接、音频转换、上报等操作；将当前共享状态（如 tts_text_buff, processed_chars）收敛到连接级上下文，避免并发写入。
- 重构 TTS/ASR 报表逻辑：_report_worker 改为 async def _report_loop()，使用异步队列和 asyncio.to_thread 调用 report()；将 report_thread 替换为单个后台异步任务，生命周期由 stop_event 控制。
- 构建统一的资源管理器模块（如 core/runtime/runtime_manager.py），负责初始化全局线程池、事件循环任务注册、系统级监控钩子；ConnectionHandler 在进入/退出时向管理器登记，便于后续扩展为多进程或分布式部署。

若需进一步扩展，可将语音编解码迁移到独立微服务或使用 asyncio.subprocess 处理外部二进制，单机只保留轻量调度；同时探索使用 Uvicorn/FastAPI 等 async-first 框架替换现有 websockets 手写服务器，最大化复用成熟事件循环。
后续建议

先原地验证短期方案的线程数下降效果，确认监控数据可靠，再规划长期重构的迭代里程碑（例如：阶段一共享线程池、阶段二 TTS 协程化、阶段三 ASR 协程化与报表重构）。
开发过程中保持单元测试/集成测试覆盖，尤其是异步与线程池交互的边界条件，避免引入新的竞态或资源泄露。

# 混合模式选择的猜想
早期设计权衡 (2025年5月)：
- 提交 5be65216 "优化线程" 显示团队试图解决线程管理问题
- 当时的"优化"主要是将线程生命周期与连接绑定，使用 conn.stop_event 控制线程退出
- 但没有从根本上解决 per-connection 线程池的问题
- 集成的第三方 ASR/TTS 提供同步接口，维护者倾向用线程隔离阻塞操作，避免拖慢 asyncio 事件循环。从 ASR、TTS 层对事件循环的重复封装可以看出，他们没有深入重写为纯异步。
- Git 记录围绕“避免阻塞”展开：885b7a0b05bcfcac46ff05bedc6b468adbd70dcc（“feat: 增加上报线程池”）又新增一个执行器让上报离线跑，直到 631787a4f12aa769226893f04cacaec180e29056 才把它并回去。没有提交说明提到连接数目标。

设计思路的可能考量：
1. 隔离性优先：每个连接独立的线程池确保一个连接的阻塞不影响其他连接
2. 简化实现：避免复杂的线程池共享和竞争问题
3. 快速原型：在项目早期，快速实现功能比最优性能更重要

技术债务积累：
- 项目混合使用 asyncio 和 threading，重构成本高
- TTS/ASR 提供商大多是同步 API，改协程需要大量适配工作
- 音频处理涉及 FFmpeg 等外部库，协程化复杂度高

性能考量误判：
- 可能认为 CPU 密集型的音频处理适合线程而非协程
- 低估了实际部署时的连接数量规模

# 第三方接口与协程化改造
## 可行性判断

- 现有“流式”供应商（例：aliyun_stream）本就用 websockets 和 asyncio.create_task 建连与推送结果，纯协程化没有接口障碍；参见 main/xiaozhi-server/core/providers/asr/aliyun_stream.py:157, main/xiaozhi-server/core/providers/asr/aliyun_stream.py:164, main/xiaozhi-server/core/providers/asr/aliyun_stream.py:175。
- 走纯 HTTP 的供应商（例：aliyun TTS）虽在 text_to_speak 里用阻塞的 requests.post（main/xiaozhi-server/core/providers/tts/aliyun.py:166, main/xiaozhi-server/core/providers/tts/aliyun.py:184），但本质只是 REST/JSON；改成 httpx.AsyncClient.post 或 aiohttp 后即可直接 await，无需专用线程。
- 本地模型类（如 vosk.py, sherpa_onnx_local.py）仍是同步函数，但完全可以通过 loop.run_in_executor(shared_pool, …) 复用全局线程池；重点是把 TTSProviderBase/ASRProviderBase 的调度逻辑换成向共享池投递，而不是 per-connection 新池。
## 潜在注意事项

- 部分官方 SDK 仍是同步封装（如 token 刷新里的 requests.get，main/xiaozhi-server/core/providers/tts/aliyun.py:74、main/xiaozhi-server/core/providers/asr/aliyun_stream.py:61），需要自行提供异步包装或放在共享池里执行。
- 要确认各供应商客户端是否线程安全；如果不是，就在共享线程池里维持“池级实例 + 互斥”即可。
- 协程化后仍要给 CPU/IO 密集段打到共享池，避免把事件循环阻塞；同时保证当前的 queue.Queue/threading.Event 通信被 asyncio.Queue/asyncio.Event 替换。
## 结论

- 第三方 ASR/TTS 接口本身没有阻止协程化和共享线程池的技术门槛：流式协议天然适配异步，HTTP/REST 可以换成异步 HTTP 客户端，本地模型可通过共享池托管。
- 重构主要工作量在于：改写基础类的任务调度/通信模型、替换阻塞网络调用、并补上线程安全和错误处理。完成这些后，就能落实协程化 + 全局线程池的目标。