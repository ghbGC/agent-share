> [HKUDS/nanobot: "🐈 nanobot: The Ultra-Lightweight Personal AI Agent"](https://github.com/HKUDS/nanobot)
> 基于 releases 版本：**v0.1.4.post5**
## nanobot Agent 系统架构总结
### 整体架构（数据流）
```
用户消息 → Channel → MessageBus(inbound) → AgentLoop → LLM Provider
                                              ↓
                                         Tool 执行
                                              ↓
                                    MessageBus(outbound) → Channel → 用户
```
### 核心模块一览
| 模块 | 文件 | 职责 |
|------|------|------|
| **AgentLoop** | `agent/loop.py` | 核心引擎：收消息→构建上下文→调LLM→执行工具→返回响应 |
| **ContextBuilder** | `agent/context.py` | 组装 system prompt + 消息列表（身份、引导文件、记忆、技能） |
| **MemoryStore** | `agent/memory.py` | 两层记忆：MEMORY.md（长期事实）+ HISTORY.md（可搜索日志） |
| **MemoryConsolidator** | `agent/memory.py` | 上下文压缩策略：token 超限时自动归档旧消息 |
| **SkillsLoader** | `agent/skills.py` | 技能发现与加载（workspace 优先，builtin 兜底） |
| **SubagentManager** | `agent/subagent.py` | 后台子 agent 执行，完成后通过 bus 通知主 agent |
| **ToolRegistry** | `agent/tools/registry.py` | 工具注册中心：注册、校验、执行 |
| **Tool(ABC)** | `agent/tools/base.py` | 工具基类：schema 定义、参数校验、类型转换 |
| **MessageBus** | `bus/queue.py` | 异步消息队列，解耦 channel 和 agent |
| **SessionManager** | `session/manager.py` | 会话持久化（JSONL 格式），支持归档偏移量 |
| **LLMProvider** | `providers/base.py` | LLM 抽象层：统一接口 + 重试 + 错误处理 |
| **CronService** | `cron/service.py` | 定时任务调度（cron 表达式 / 一次性 / 间隔） |
| **HeartbeatService** | `heartbeat/service.py` | 周期性自检（读 HEARTBEAT.md，LLM 判断是否有任务） |
| **Evaluator** | `utils/evaluator.py` | 后台任务结果评估（是否需要通知用户） |
### 关键执行流程
**1. 消息处理主流程** (`AgentLoop._process_message`)
```
收到消息 → 获取/创建 Session → 检查是否需要上下文压缩
→ 设置工具上下文(channel/chat_id) → 构建消息列表(system+history+user)
→ 进入 agent 循环 → 保存本轮消息到 session → 后台触发压缩检查
```
**2. Agent 循环** (`AgentLoop._run_agent_loop`)
```
while iteration < max_iterations:
    调 LLM（带 tool definitions）
    ├─ 有 tool_calls → 逐个执行工具 → 结果追加到 messages → 继续循环
    └─ 无 tool_calls → 返回最终文本 → 结束
```
**3. 上下文构建** (`ContextBuilder.build_system_prompt`)
```
身份信息（nanobot 🐈 + 运行时 + workspace 路径）
+ 引导文件（AGENTS.md, SOUL.md, USER.md, TOOLS.md）
+ 长期记忆（MEMORY.md）
+ always 技能（自动加载的 SKILL.md）
+ 技能摘要（XML 格式，按需 read_file 加载）
```
**4. 记忆压缩** (`MemoryConsolidator.maybe_consolidate_by_tokens`)
```
估算当前 prompt token 数
→ 超过 context_window_tokens → 找到用户轮次边界
→ 取出旧消息块 → 调 LLM 总结 → 写入 HISTORY.md + 更新 MEMORY.md
→ 更新 session.last_consolidated 偏移量
→ 循环直到 token 降到目标值（context_window / 2）
```
**5. 子 Agent** (`SubagentManager`)
```
spawn(task) → 创建独立 asyncio.Task
→ 构建独立工具集（无 message/spawn 工具）
→ 运行独立 agent 循环（最多 15 轮）
→ 完成后通过 bus 注入 system 消息 → 主 agent 自然语言总结后通知用户
```
---
## 阅读清单（建议阅读顺序）
### 第一阶段：理解核心骨架
- [ ] **1. MessageBus** (`bus/queue.py`, 44行) — 最简单的模块，理解 inbound/outbound 双队列设计
- [ ] **2. 事件类型** (`bus/events.py`, 38行) — InboundMessage / OutboundMessage 数据结构
- [ ] **3. Session** (`session/manager.py`) — Session 数据结构 + JSONL 持久化 + `get_history()` 的合法边界对齐逻辑
- [ ] **4. Tool 基类** (`agent/tools/base.py`) — 抽象接口 + JSON Schema 校验 + 类型转换
- [ ] **5. ToolRegistry** (`agent/tools/registry.py`) — 注册/查找/执行，注意错误处理策略
### 第二阶段：理解上下文与记忆
- [ ] **6. ContextBuilder** (`agent/context.py`) — system prompt 拼接逻辑，理解各部分优先级
- [ ] **7. MemoryStore** (`agent/memory.py` 上半部分) — 两层记忆读写 + `consolidate()` 用 LLM 做总结
- [ ] **8. MemoryConsolidator** (`agent/memory.py` 下半部分) — token 估算 + 边界选择 + 循环压缩策略
- [ ] **9. SkillsLoader** (`agent/skills.py`) — 技能发现（workspace > builtin）、frontmatter 解析、依赖检查
### 第三阶段：理解核心引擎
- [ ] **10. AgentLoop** (`agent/loop.py`) — 重点看：
  - `run()` — 主事件循环 + /stop /restart 命令处理
  - `_dispatch()` — 全局锁 + 异步分发
  - `_process_message()` — 完整消息处理流程
  - `_run_agent_loop()` — LLM 调用 + 工具执行循环
  - `_save_turn()` — 消息持久化 + 截断 + 清理
### 第四阶段：理解扩展系统
- [ ] **11. SubagentManager** (`agent/subagent.py`) — 独立工具集 + 独立循环 + bus 回报机制
- [ ] **12. LLMProvider** (`providers/base.py`) — 抽象接口 + `chat_with_retry` 重试策略 + 图片降级
- [ ] **13. CronService** (`cron/service.py`) — 三种调度模式 + 定时器 + 磁盘持久化
- [ ] **14. HeartbeatService** (`heartbeat/service.py`) — 两阶段设计（决策→执行）+ LLM 工具调用判断
- [ ] **15. Evaluator** (`utils/evaluator.py`) — 通知门控，避免无意义打扰
### 第五阶段：深入具体工具实现
- [ ] **16. 文件工具** (`agent/tools/filesystem.py`) — read/write/edit/list_dir，注意 allowed_dir 安全限制
- [ ] **17. Shell 工具** (`agent/tools/shell.py`) — 命令执行 + 超时 + 工作目录限制
- [ ] **18. Web 工具** (`agent/tools/web.py`) — search + fetch，代理支持
- [ ] **19. Message 工具** (`agent/tools/message.py`) — 跨通道消息发送
- [ ] **20. Spawn 工具** (`agent/tools/spawn.py`) — 子 agent 触发入口
- [ ] **21. Cron 工具** (`agent/tools/cron.py`) — agent 自主管理定时任务
- [ ] **22. MCP 工具** (`agent/tools/mcp.py`) — Model Context Protocol 外部工具集成
### 第六阶段：配置与辅助
- [ ] **23. 配置加载** (`config/loader.py` + `config/schema.py`) — config.json 解析
- [ ] **24. 辅助函数** (`utils/helpers.py`) — token 估算、时间格式化等工具函数