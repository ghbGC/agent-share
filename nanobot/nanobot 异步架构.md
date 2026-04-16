## 一、消息总线（MessageBus）
### 1.1 本质
MessageBus 就是**两个 asyncio.Queue**，非常简单：
```python
class MessageBus:
    def __init__(self):
        self.inbound: asyncio.Queue[InboundMessage]   # 入站队列（用户 → agent）
        self.outbound: asyncio.Queue[OutboundMessage]  # 出站队列（agent → 用户）
```
它是一个**中介者**，让 Channel 和 AgentLoop 互相不知道对方的存在。
### 1.2 为什么需要消息总线？
如果没有 MessageBus，AgentLoop 需要直接调用 `feishu_channel.send()`，Channel 需要直接调用 `agent_loop.process()`。这样：
- AgentLoop 依赖了具体的 Channel 实现
- 加新 Channel 要改 AgentLoop 代码
- 无法支持多 Channel 同时运行
有了 MessageBus：
- AgentLoop 只管从 inbound 取消息、往 outbound 放消息
- Channel 只管往 inbound 放消息、从 outbound 取消息
- 两者完全解耦
### 1.3 完整的消息流转路径
以飞书为例，一条消息的完整旅程：
```
飞书服务器
  │ (WebSocket 推送)
  ▼
FeishuChannel._on_message()          ← 飞书 SDK 回调（在 WebSocket 线程中）
  │ asyncio.run_coroutine_threadsafe()
  ▼
FeishuChannel._on_message()          ← 切到主事件循环
  │ _handle_message(sender_id, chat_id, content, media, metadata)
  ▼
BaseChannel._handle_message()        ← 权限检查
  │ bus.publish_inbound(InboundMessage(...))
  ▼
MessageBus.inbound.put()             ← 消息入队
  │
  ▼
AgentLoop.run()                      ← 主循环
  │ bus.consume_inbound(timeout=1.0)
  ▼
MessageBus.inbound.get()             ← 消息出队
  │
  ▼
AgentLoop._dispatch() → _process_message() → _run_agent_loop()
  │ ... LLM 调用 + 工具执行 ...
  │ bus.publish_outbound(OutboundMessage(...))
  ▼
MessageBus.outbound.put()            ← 响应入队
  │
  ▼
ChannelManager._dispatch_outbound()  ← 出站分发循环
  │ bus.consume_outbound(timeout=1.0)
  ▼
MessageBus.outbound.get()            ← 响应出队
  │ channel = self.channels.get(msg.channel)
  │ channel.send(msg)
  ▼
FeishuChannel.send()                 ← 调飞书 API 发消息
  │
  ▼
飞书服务器 → 用户手机
```
### 1.4 谁在消费 outbound？
**Gateway 模式**（`nanobot gateway`）：
```python
# commands.py 第 603-610 行
async def run():
    await asyncio.gather(
        agent.run(),           # 协程1：AgentLoop 主循环
        channels.start_all(),  # 协程2：ChannelManager（内含 outbound 分发）
    )
```
`channels.start_all()` 内部启动了 `_dispatch_outbound()` 协程，它不断从 `bus.outbound` 取消息，根据 `msg.channel` 路由到对应的 Channel。
**CLI 模式**（`nanobot chat`）：
```python
# commands.py 第 733-810 行
bus_task = asyncio.create_task(agent_loop.run())        # 协程1
outbound_task = asyncio.create_task(_consume_outbound()) # 协程2
```
CLI 模式没有 ChannelManager，自己写了一个 `_consume_outbound()` 来消费 outbound 消息，直接打印到终端。
### 1.5 InboundMessage 的 session_key
```python
@property
def session_key(self) -> str:
    return self.session_key_override or f"{self.channel}:{self.chat_id}"
```
`session_key` 是会话的唯一标识，格式 `"channel:chat_id"`。比如：
- 飞书私聊：`"feishu:ou_xxx"`
- Telegram：`"telegram:xxx"`
- CLI：`"cli:direct"`
AgentLoop 用它来区分不同用户/会话的 Session。
---
## 二、异步执行流程
### 2.1 Gateway 模式的协程拓扑
```
asyncio.run(run())
  │
  └─ asyncio.gather(
       │
       ├─ agent.run()                    ← 协程 A：AgentLoop 主循环
       │   │
       │   └─ while self._running:
       │       msg = bus.consume_inbound(timeout=1.0)
       │       task = asyncio.create_task(_dispatch(msg))  ← 创建子任务
       │
       ├─ channels.start_all()           ← 协程 B：ChannelManager
       │   │
       │   ├─ asyncio.gather(
       │   │   feishu_channel.start(),    ← 协程 B1：飞书 WebSocket 循环
       │   │   telegram_channel.start(),  ← 协程 B2：Telegram polling 循环
       │   │   ...
       │   │ )
       │   │
       │   └─ _dispatch_outbound()       ← 协程 B3：出站分发循环
       │       while True:
       │         msg = bus.consume_outbound(timeout=1.0)
       │         channel.send(msg)
       │
       ├─ cron.start()                   ← 协程 C：定时任务
       │   └─ while self._running:
       │       await asyncio.sleep(delay)
       │       await self._on_timer()
       │
       └─ heartbeat.start()              ← 协程 D：心跳检查
           └─ while self._running:
               await asyncio.sleep(interval)
               await self._tick()
```
**所有这些协程都在同一个 asyncio 事件循环中并发运行**，由 asyncio 的调度器在它们之间切换。
### 2.2 asyncio 事件循环的核心概念
**事件循环**就像一个餐厅只有一个厨师（单线程），但可以同时做多道菜（协程）：
```
时间线 ──────────────────────────────────────────────→
协程A (AgentLoop):
  consume_inbound() ──等待──→ 收到消息 → _dispatch() ──等待──→ 收到消息 → ...
协程B1 (飞书WebSocket):
  ws.start() ──等待WebSocket数据──→ 收到消息 → _handle_message() ──等待──→ ...
协程B3 (出站分发):
  consume_outbound() ──等待──→ 收到响应 → feishu.send() ──等待──→ ...
协程C (Cron):
  sleep(300s) ───────────────────────────────→ _on_timer() → sleep(300s) → ...
```
关键点：
- **同一时刻只有一个协程在执行**（单线程）
- **`await` 是切换点**：遇到 `await` 就让出控制权，事件循环去执行其他协程
- **`asyncio.Queue.get()` 是阻塞等待**：队列为空时协程挂起，不占 CPU
- **`asyncio.sleep()` 也是挂起**：不占 CPU，到时间后事件循环唤醒它
### 2.3 `asyncio.create_task` vs `asyncio.gather`
**`asyncio.create_task(coro)`**：把协程包装成 Task，**立即开始调度**，不等待完成。返回 Task 对象，可以用来取消。
```python
# AgentLoop.run() 中
task = asyncio.create_task(self._dispatch(msg))
# _dispatch 开始在后台运行，run() 继续下一次循环
```
**`asyncio.gather(*coros)`**：同时启动多个协程，**等待全部完成**。
```python
# Gateway 启动时
await asyncio.gather(agent.run(), channels.start_all())
# 两个都运行，直到其中一个抛异常或被取消
```
### 2.4 AgentLoop 的任务模型
```
AgentLoop.run()  ← 主协程，永远运行
  │
  │  收到消息
  │
  ├─ asyncio.create_task(_dispatch(msg_1))  ← Task 1
  │   └─ async with _processing_lock:       ← 等锁
  │       └─ _process_message(msg_1)        ← 处理中...
  │
  ├─ asyncio.create_task(_dispatch(msg_2))  ← Task 2（排队等锁）
  │   └─ async with _processing_lock:       ← 等锁...
  │
  └─ asyncio.create_task(_dispatch(msg_3))  ← Task 3（排队等锁）
```
**虽然创建了多个 Task，但全局锁保证串行执行**：
```
时间线 ──────────────────────────────────────────────→
Task 1:  [====处理消息1====]
Task 2:                      [等待锁][====处理消息2====]
Task 3:                                           [等待锁][====处理消息3====]
```
### 2.5 全局锁 `_processing_lock` 的作用
```python
self._processing_lock = asyncio.Lock()
async def _dispatch(self, msg):
    async with self._processing_lock:  # 获取锁
        response = await self._process_message(msg)
        ...
```
为什么需要锁？假设没有锁：
```
用户快速发了两条消息 A 和 B
Task A: 读 session → 构建 prompt → 调 LLM → 保存 session
Task B:           读 session（此时 A 还没保存）→ 构建 prompt → 调 LLM → 保存 session（覆盖了 A 的结果！）
```
有了锁：
```
Task A: [读 session → 构建 prompt → 调 LLM → 保存 session]
Task B: [等锁...........................................][读 session → ...]
```
B 拿到的 session 是 A 处理完之后的，不会丢失 A 的消息。
### 2.6 `/stop` 如何打断正在执行的任务
```python
async def _handle_stop(self, msg):
    tasks = self._active_tasks.pop(msg.session_key, [])
    cancelled = sum(1 for t in tasks if not t.done() and t.cancel())
    for t in tasks:
        await t  # 等待 CancelledError 传播
```
`task.cancel()` 会在 Task 当前 `await` 的位置注入 `CancelledError`：
```
Task 1 正在执行:
  _process_message()
    → _run_agent_loop()
      → provider.chat_with_retry()  ← await 这里
        → asyncio.sleep(2)          ← retry 等待
/stop 触发:
  task.cancel()
  → CancelledError 注入到 asyncio.sleep(2) 处
  → _run_agent_loop() 不捕获 CancelledError
  → _dispatch() 不捕获 CancelledError（只捕获 Exception）
  → CancelledError 传播到 _handle_stop() 的 await t
```
注意 `_dispatch` 中的异常处理：
```python
except asyncio.CancelledError:
    raise  # ← 不吞！让 /stop 能正常取消
except Exception:
    # 其他异常才返回错误消息
```
### 2.7 进度消息的异步发送
agent 循环中，每次工具调用前会发送进度：
```python
# _run_agent_loop 中
if response.has_tool_calls:
    if on_progress:
        await on_progress(response.content)     # LLM 的思考
        await on_progress(tool_hint, tool_hint=True)  # 工具调用提示
```
`on_progress` 实际上是：
```python
async def _bus_progress(content, *, tool_hint=False):
    await self.bus.publish_outbound(OutboundMessage(
        channel=msg.channel, chat_id=msg.chat_id,
        content=content, metadata={"_progress": True, "_tool_hint": tool_hint},
    ))
```
这个 `await` 不会阻塞——它只是把消息放到 outbound 队列里就返回了。真正的发送由 `_dispatch_outbound()` 协程异步完成。
```
AgentLoop (_run_agent_loop):
  调 LLM → 有工具调用 → publish_outbound(进度) → 执行工具 → 调 LLM → ...
ChannelManager (_dispatch_outbound):
  consume_outbound() → feishu.send(进度) → consume_outbound() → feishu.send(最终响应) → ...
```
两者并行运行，进度消息和最终响应都能及时发送。
### 2.8 子 agent 的异步机制
```python
# SubagentManager.spawn()
bg_task = asyncio.create_task(self._run_subagent(task_id, task, ...))
```
子 agent 是一个独立的 asyncio.Task，和主 agent 并行运行。但子 agent **没有 message 工具和 spawn 工具**（不能发消息、不能再生子 agent）。
子 agent 完成后，通过 bus 注入一条 system 消息：
```python
# SubagentManager._announce_result()
msg = InboundMessage(
    channel="system",
    sender_id="subagent",
    chat_id=f"{origin['channel']}:{origin['chat_id']}",
    content=announce_content,
)
await self.bus.publish_inbound(msg)
```
这条消息进入 inbound 队列，被 AgentLoop.run() 取出后，走正常的 `_dispatch` → `_process_message` 流程。主 agent 会看到子 agent 的结果，自然语言总结后发给用户。
### 2.9 后台任务的异步机制
```python
def _schedule_background(self, coro):
    task = asyncio.create_task(coro)
    self._background_tasks.append(task)
    task.add_done_callback(self._background_tasks.remove)
```
记忆压缩等操作被包装成后台 Task，不阻塞用户响应。它们在事件循环空闲时执行。
### 2.10 飞书 WebSocket 的线程模型
飞书 SDK 的 WebSocket 运行在**单独的线程**中（不是 asyncio 协程）：
```python
# FeishuChannel.start()
def run_ws():
    ws_loop = asyncio.new_event_loop()  # 新建事件循环
    asyncio.set_event_loop(ws_loop)
    while self._running:
        self._ws_client.start()  # 阻塞调用
self._ws_thread = threading.Thread(target=run_ws, daemon=True)
self._ws_thread.start()
```
收到消息后，通过 `asyncio.run_coroutine_threadsafe()` 切回主事件循环：
```python
def _on_message_sync(self, data):
    if self._loop and self._loop.is_running():
        asyncio.run_coroutine_threadsafe(self._on_message(data), self._loop)
```
这是因为飞书 SDK 的 WebSocket 回调在子线程中，但 nanobot 的所有异步操作都在主事件循环中。`run_coroutine_threadsafe` 是线程安全的桥梁。

---
## 三、完整时序图（用户发一条消息）
```
时间 ──────────────────────────────────────────────────────────→
线程1 (主事件循环):
  │
  │  ┌─ AgentLoop.run() ──────────────────────────────────────┐
  │  │  consume_inbound() ──等待──────────────────→ 取到消息  │
  │  │  create_task(_dispatch(msg))                          │
  │  │  consume_inbound() ──等待──────────────────→ ...       │
  │  └───────────────────────────────────────────────────────┘
  │
  │  ┌─ _dispatch(msg) ──────────────────────────────────────┐
  │  │  async with _processing_lock:                         │
  │  │    _process_message(msg)                              │
  │  │      ├─ session.get_or_create()                       │
  │  │      ├─ context.build_messages()                      │
  │  │      ├─ _run_agent_loop()                             │
  │  │      │   ├─ LLM 调用 ──await──→                        │
  │  │      │   ├─ publish_outbound(进度) ──await──→          │
  │  │      │   ├─ tools.execute() ──await──→                 │
  │  │      │   └─ LLM 调用 ──await──→                        │
  │  │      ├─ _save_turn()                                  │
  │  │      └─ publish_outbound(最终响应) ──await──→          │
  │  └───────────────────────────────────────────────────────┘
  │
  │  ┌─ _dispatch_outbound() ────────────────────────────────┐
  │  │  consume_outbound() ──等待──→ 取到进度 → feishu.send()│
  │  │  consume_outbound() ──等待──→ 取到响应 → feishu.send()│
  │  └───────────────────────────────────────────────────────┘
  │
线程2 (飞书 WebSocket):
  │
  │  ws_client.start() ──等待WebSocket数据──→ 收到消息
  │  _on_message_sync(data)
  │  run_coroutine_threadsafe(_on_message(data), main_loop)
  │  ws_client.start() ──等待WebSocket数据──→ ...
```
