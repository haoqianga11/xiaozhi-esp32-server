# 并发与线程检查清单（由简到繁、相关项聚合）

## 全局入口与基础异步任务（最简单）
- [ ] main/xiaozhi-server/app.py
  - [ ] monitor_stdin 异步任务已启动/退出正确（回车消费、取消与回收）
  - [ ] WebSocketServer.start 启动正常（端口/地址、异常处理）
  - [ ] SimpleHttpServer.start 启动正常（路由、长驻循环、异常处理）
  - [ ] wait_for_exit 信号处理与任务取消顺序正确（stdin_task、ws_task、ota_task）
- [ ] main/xiaozhi-server/config/logger.py
  - [ ] 日志配置 enqueue=True 产生后台日志线程（库内部线程，知晓其存在）
  - [ ] 控制台/文件输出格式与等级符合预期

## 连接生命周期与路由（异步骨架）
- [ ] main/xiaozhi-server/core/websocket_server.py
  - [ ] _handle_connection 为每个连接创建独立 ConnectionHandler
  - [ ] update_config 使用 asyncio.Lock 保护，模块热更新路径正确
- [ ] main/xiaozhi-server/core/connection.py
  - [ ] handle_connection 头部解析、认证、device-id/client-id 获取
  - [ ] 超时任务 _check_timeout 创建、退出、日志正确
  - [ ] 私有配置 _initialize_private_config 合并粒度/模块实例化标志位正确
  - [ ] 路由 _route_message 文本/音频分发正确

## 每连接的常驻线程（逐项确认）
- [ ] 线程池（5 个）
  - [ ] ThreadPoolExecutor(max_workers=5) 已创建
  - [ ] 用途：_initialize_components、_process_report 及其他阻塞型工作
- [ ] ASR 消费线程（1 个）
  - [ ] core/providers/asr/base.py: open_audio_channels -> asr_priority_thread
  - [ ] 职责：从 conn.asr_audio_queue 取数据，run_coroutine_threadsafe 调用 handleAudioMessage
- [ ] TTS 文本线程（1 个）
  - [ ] core/providers/tts/base.py: tts_text_priority_thread
  - [ ] 职责：文本分段、to_tts_stream、打断处理
- [ ] TTS 音频发送线程（1 个）
  - [ ] core/providers/tts/base.py: _audio_play_priority_thread
  - [ ] 职责：流控发送、设备消费模拟、上报聚合
- [ ] 上报线程（0/1 个，按配置）
  - [ ] core/connection.py: _init_report_threads -> report_thread
  - [ ] 触发条件：read_config_from_api 且 chat_history_conf > 0

## 临时/场景性线程（易遗漏，发生在特定阶段）
- [x] 记忆保存线程（关闭时）
  - [x] core/connection.py: _save_and_close -> threading.Thread(save_memory_task, daemon=True)
  - [x] 独立事件循环 + await memory.save_memory，异常捕获、最终关闭
  - ✅ 实现正确：创建新事件循环，在独立线程中保存记忆，异常处理完善，确保loop.close()
- [x] 服务重启线程（收到重启指令时）
  - [x] core/connection.py: handle_restart -> threading.Thread(restart_server, daemon=True)
  - [x] Popen 重启 app.py + os._exit(0) 语义确认
  - ✅ 实现正确：使用daemon线程，Popen创建新进程，start_new_session=True，最后os._exit(0)
- [x] 声纹并发短时线程池（语音停止聚合处理）
  - [x] core/providers/asr/base.py: handle_voice_stop -> ThreadPoolExecutor(max_workers=2)
  - [x] 并行执行最终 ASR 与声纹识别；超时、异常与结果合流处理
  - ✅ 实现正确：独立事件循环，并行执行ASR和声纹识别，15s超时，异常处理完善

## 流式 ASR/TTS 的异步 forward/monitor 任务（非线程，但关键）
- [ ] ASR 流式转发任务
  - [ ] aliyun_stream.ASRProvider: forward_task = asyncio.create_task(_forward_results)
  - [ ] doubao_stream.ASRProvider: forward_task = asyncio.create_task(_forward_asr_results)
  - [ ] 断线重连/错误码与静音处理路径检查
- [ ] TTS 流式/双流式监听任务
  - [ ] aliyun_stream.TTSProvider: _monitor_task = asyncio.create_task(_start_monitor_tts_response)
  - [ ] huoshan_double_stream.TTSProvider: _monitor_task = asyncio.create_task(_start_monitor_tts_response)
  - [ ] start_session/finish_session 生命周期与 _monitor_task 收尾

## 队列/流控/上报链路（数据面）
- [ ] 队列完整性
  - [ ] ASR：conn.asr_audio_queue（生产：_route_message；消费：asr_priority_thread）
  - [ ] TTS：self.tts_text_queue、self.tts_audio_queue（双线程协同）
  - [ ] 上报：self.report_queue（report_thread 消费，线程池执行 _process_report）
- [ ] 关闭时清空队列
  - [ ] core/connection.py: clear_queues 非阻塞清空，顺序/异常处理
- [ ] 流控
  - [ ] core/utils/audio_flow_control.py：FlowControlConfig 默认值、TokenBucket 行为
  - [ ] TTS 发送线程：can_send_frames/wait 超时策略、设备消费更新时机
- [ ] 音频发送链
  - [ ] core/handle/sendAudioHandle.py：sendAudioMessage 调用链正确（句首/中/尾、文本与音频同步）

## 可选能力与依赖（跨模块）
- [ ] 声纹识别能力
  - [ ] core/utils/voiceprint_provider.py：URL/key/speakers 解析、健康检查缓存
  - [ ] 启用条件：config.voiceprint 存在；连接时 initialize；handle_voice_stop 聚合识别
- [ ] 记忆模块
  - [ ] memory.init_memory 参数（role_id/llm/summaryMemory/save_to_file）
  - [ ] mem_local_short 模式的专用 LLM 创建与 set_llm，或复用主 LLM
- [ ] 意图与工具
  - [ ] intent 类型：nointent/function_call/intent_llm 选择与专用 LLM
  - [ ] UnifiedToolHandler 初始化与 cleanup 调用
- [ ] MCP/IoT/插件
  - [ ] 仅异步/网络 IO，无显式常驻线程；资源清理路径确认

## 关闭与资源回收（统一收尾）
- [ ] core/connection.py: close
  - [ ] 取消 timeout_task；func_handler.cleanup；stop_event.set
  - [ ] clear_queues；WebSocket 安全关闭（ws/state 检测）
  - [ ] tts.close；executor.shutdown(wait=False)；日志与异常捕获
- [ ] TTS 会话收尾
  - [ ] finish_session 的发送/等待监听任务完成
- [ ] ASR 收尾
  - [ ] stop_ws_connection / _cleanup（各 provider 内部）在连接关闭/错误时触发

## 配置热更新与模块动态替换
- [ ] core/websocket_server.py: update_config
  - [ ] 获取新配置、检测 VAD/ASR 更新、initialize_modules 带标志位
  - [ ] 组件实例替换与并发访问安全性（config_lock）
  - [ ] 日志与错误处理完善

## 并发资源盘点（逐连接维度，确认数量）
- [ ] 线程池：5
- [ ] ASR 线程：1
- [ ] TTS 线程：2（文本 + 音频）
- [ ] 上报线程：0/1（按配置）
- [ ] 临时线程：0/1（记忆保存）+ 0/1（重启）
- [ ] 短时线程池：最多 2（声纹并发）
- [ ] 流式异步任务：0~2（ASR forward）+ 0~1（TTS monitor）

## 性能/压测与参数
- [ ] main/xiaozhi-server/performance_tester/* 可运行/参数可配置
- [ ] config.yaml 中 TTS/ASR/VAD 关键参数、超时与速率对齐（与流控一致）
- [ ] 线程增长预期与上限（并发连接 N 下的线程总数）

## 潜在改进点（选做项）
- [ ] 将每连接线程池切换为全局共享池或分组池，控制总线程数
- [ ] 将上报线程改为 asyncio 任务或复用线程池，减少每连接常驻线程
- [ ] 为各线程/任务添加命名与健康探针、度量指标（队列长度、等待时长、丢弃计数）
- [ ] 完善错误码与重试策略（流式 ASR/TTS 的网络抖动/限流）

---

### 附：关键文件定位（便于对照）
- [ ] main/xiaozhi-server/app.py
- [ ] main/xiaozhi-server/core/websocket_server.py
- [ ] main/xiaozhi-server/core/http_server.py
- [ ] main/xiaozhi-server/core/connection.py
- [ ] main/xiaozhi-server/core/providers/asr/base.py
- [ ] main/xiaozhi-server/core/providers/asr/aliyun_stream.py
- [ ] main/xiaozhi-server/core/providers/asr/doubao_stream.py
- [ ] main/xiaozhi-server/core/providers/tts/base.py
- [ ] main/xiaozhi-server/core/providers/tts/aliyun_stream.py
- [ ] main/xiaozhi-server/core/providers/tts/huoshan_double_stream.py
- [ ] main/xiaozhi-server/core/utils/audio_flow_control.py
- [ ] main/xiaozhi-server/core/handle/sendAudioHandle.py
- [ ] main/xiaozhi-server/core/utils/voiceprint_provider.py

# 退出信号的层级传播
1. [Ctrl+C / kill 信号]
   ↓
2. [wait_for_exit() 返回]
   ↓  
3. [app.py 的 finally 块执行]
   ↓
4. [ws_task.cancel() - 取消WebSocket服务器任务]
   ↓
5. [WebSocket服务器关闭，停止接受新连接]
   ↓
6. [所有 active_connections 的WebSocket连接断开]
   ↓
7. [每个ConnectionHandler的handle_connection异常退出]
   ↓
8. [触发ConnectionHandler.close()方法]
   ↓
9. [stop_event.set() - 设置停止事件]
   ↓
10. [所有线程检查stop_event.is_set()并退出]
    ├── ASR消费线程退出
    ├── TTS文本线程退出  
    ├── TTS音频线程退出
    ├── 上报线程退出
    └── 其他异步任务退出
# 图
```mermaid

```