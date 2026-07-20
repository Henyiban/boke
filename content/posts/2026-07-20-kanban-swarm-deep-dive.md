---
title: "Kanban Swarm：Hermes 的多智能体协作引擎深度解析"
date: 2026-07-20
tags:
  - kanban
  - 多智能体
  - Hermes Agent
  - AI Agent
  - 任务调度
  - Swarm
author: "Henyiban"
---

> 目标读者：AI 工程师 / Agent 架构师 / 多智能体系统开发者
> 字数：约 8000 字
> 关键词：kanban、多智能体、任务调度、故障恢复、swarm、orchestration

---

## 一、当你需要一群 AI 同时干活

你跟 AI 说："帮我研究一下 ICP 融资数据，北美和欧洲市场分开查，然后合成一篇分析稿。"

如果对面只有一个 agent，它会串行做完所有事情——先查北美，再查欧洲，然后自己写。慢不说，你还不能中途插手看进度。

如果你有三个 agent——两个研究员、一个写手——它们可以并行工作，研究员们各自查各自的市场，写完交接给写手。中间任何一个环节卡住了，你可以看到是谁在等什么，手动介入。

这就是 Hermes Kanban 设计要解决的问题。

它不是让你跟一个 AI 聊天时临时 spawn 一个子进程（那是 `delegate_task` 的活）。它是一个**持久化的任务板**，放在 `~/.hermes/kanban.db` 里，让多个命名 profile——你可以理解成不同角色的"数字分身"——在上面认领、执行、交接任务。每次交接都是一条 SQLite 行；每个 worker 是一个独立的 OS 进程。挂掉了可以重来，崩溃了可以回收，人随时可以评论区插话。

写这篇文章的时候，Kanban 刚从 v1 早期迭代稳定下来。核心的 `kanban_db.py` 有 332KB，是一个相当扎实的状态机实现。本文会覆盖架构全景、核心机制、失效处理体系、Swarm 拓扑、竞品对比和实践建议。如果你在考虑用多 agent 系统做生产级工作流，这里有你需要的所有技术细节。

---

## 二、两种范式：`delegate_task` vs Kanban

先说清它们不是同一个东西。

`delegate_task` 是一个函数调用。你 spawn 一个子 agent，给它一段 prompt，它干完活把结果塞回你的上下文。同步的，一次性的，agent 挂了就挂了。

Kanban 是一个消息队列加状态机。你创建任务，指定 assignee（一个 profile 名），然后 fire-and-forget。Dispatcher 每隔 60 秒扫描一次板子，把 ready 的任务分配给对应 profile 的 worker 进程。任务可以 block → unblock → re-run；worker 崩溃了 dispatcher 回收后重新 dispatch。

一句话区分：**`delegate_task` 是 RPC，Kanban 是 work queue。**

| 维度 | `delegate_task` | Kanban |
|------|----------------|--------|
| 形态 | RPC（fork → join） | 持久化消息队列 + 状态机 |
| 父 agent | 阻塞等待子 agent 返回 | 创建完即 fire-and-forget |
| 子 agent 身份 | 匿名 subagent | 命名 profile，有持久化 memory |
| 可恢复性 | 无——失败即失败 | Block → unblock → re-run; crash → reclaim |
| 人类介入 | 不支持 | Comment / unblock 随时插话 |
| 单任务涉及 agent 数 | 一个调用 = 一个 subagent | N 个 agent 跨越多次 retry/review/follow-up |
| 审计追踪 | 上下文压缩后丢失 | SQLite 永久行 |
| 协调方式 | 层级式（调用者 → 被调用者） | 对等式——任何 profile 都可以读写任何 task |

两者共存。一个 kanban worker 内部可以调用 `delegate_task` 做子任务。

什么时候用哪个？

- **用 `delegate_task`**：父 agent 需要一个短推理结果才能继续，没有人类参与，结果回到父 agent 上下文中。
- **用 Kanban**：工作跨越 agent 边界，需要抵御重启，可能需要人类输入，可能由不同角色接手，或需要事后可发现。

---

## 三、架构全景：四模块 + 九工具

Kanban 系统的源码分布在四个核心模块：

### 1. `kanban_db.py`（332KB）——心脏

这是整个系统最大的单文件。包含：

- **6 张表**的数据模型：`tasks`、`task_runs`、`task_links`、`task_comments`、`task_events`、`notify_subs`
- **完整状态机**：`triage → todo → ready → running → blocked/done → archived`
- **dispatcher 循环**：每 60 秒执行 4 项扫描——回收 stale claims、回收崩溃 worker、提升 ready 任务、原子认领并 spawn
- **并发控制**：WAL 模式 + `BEGIN IMMEDIATE` + CAS（compare-and-swap）认领，SQLite 自身的序列化写者保证正确性
- **断路器**：连续失败上限（默认 2 次）后 auto-block
- **unblock 循环检测**：同一原因反复 block → unblock → re-block 达到阈值（默认 2 次）后路由到 triage，防止 cron 死循环
- **heartbeat 降级**：超过 `stale_timeout_seconds`（默认 4 小时）且最后 heartbeat 超过 1 小时，回收任务
- **protocol violation 检测**：worker 正常退出但没调 `kanban_complete` / `kanban_block`，连续 3 次后 auto-block
- **幂等创建**：通过 `idempotency_key` 防止自动化重复创建

### 2. `kanban_swarm.py`——Swarm v1 拓扑

在现有 Kanban 内核上叠加 Swarm 协作模式。当前 v1 实现了一个**三段式**管道：

```
parallel workers → verifier → synthesizer
```

- **Parallel workers**：多个 worker 并发执行同一类型的工作（比如并行研究不同市场）
- **Verifier**：等所有 parallel workers 完成后，验证它们的输出（质量检查、一致性校验）
- **Synthesizer**：将验证过的结果合成为最终产物

它们之间通过**共享黑板（shared blackboard）**通信。黑板被设计成低技术含量的结构化 JSON 注释——这样 dashboard、notifier 和任何能读 comment thread 的组件都兼容。它不是一套新的通信协议，而是利用现有的 `kanban_comment` 机制做结构化数据交换。

关键设计决策：Swarm **没有引入第二个调度器**。它只是在现有 Kanban 任务板上写任务图。Dispatcher 照常调度，task_links 照常做依赖管理。Swarm 是一个编排模式，不是一个独立的运行时。

### 3. `kanban.py`——CLI 表面

人和脚本操作板的 CLI 入口。所有 CLI 动词都有对应的 `kanban_*` tool-call 等价物。支持批量操作（`complete t_a t_b t_c`）、board 管理（多板隔离）、watch 实时事件流。

### 4. `kanban_decompose.py`——AI 分解引擎

Triage 列的 LLM 驱动分解器。读你的 profile roster（含描述），让 LLM 产出一个 JSON 任务图——哪些任务、分配给谁、谁依赖谁。用于自动 orchestration 流程。

### 九大 `kanban_*` 工具

Dispatcher spawn 的 worker 拿到的是工具，不是 CLI：

| 工具 | 用途 | 谁用 |
|------|------|------|
| `kanban_show` | 读任务全貌（title, body, 父任务 handoff, 历史尝试, 评论区） | 所有 worker |
| `kanban_list` | 按 assignee/status/tenant 列任务 | orchestrator |
| `kanban_complete` | 结束任务，写 summary + metadata 交接 | 所有 worker |
| `kanban_block` | 停住任务，按原因分类路由 | 所有 worker |
| `kanban_heartbeat` | 长操作期间发心跳 | 所有 worker |
| `kanban_comment` | 追加持久化评论 | 所有 worker |
| `kanban_create` | 拆出子任务 | orchestrator |
| `kanban_link` | 事后加依赖边 | orchestrator |
| `kanban_unblock` | 把 blocked 任务移回 ready | orchestrator |

工具而非 shell 调用的理由是硬核的：

1. **后端可移植性**。Worker 的 terminal 可能指向 Docker/Modal/SSH 容器，里面没装 hermes，也挂载不到 `~/.hermes/kanban.db`。工具在 agent 自己的 Python 进程里跑。
2. **无 shell quoting 陷阱**。`--metadata '{"files": [...]}'` 过 shlex + argparse 是 latent footgun。结构化工具参数直接跳过。
3. **更好的错误处理**。工具返回结构化 JSON，模型可以推理，而不是解析 stderr 字符串。

---

## 四、核心机制深潜

### 4.1 数据模型：6 张表驱动的状态机

Kanban 的数据模型刻意保持简洁。没有 ORM，没有 migration 框架——就是 6 张 SQLite 表和一个 Python 模块。

**`tasks` 表**——核心行。字段包括：
- `id`：`t_` 前缀的 8 位 hex
- `title`，`body`（markdown）
- `assignee`：profile 名称
- `status`：7 种状态之一
- `tenant`：可选的命名空间字符串
- `idempotency_key`：自动化去重
- `priority`：dispatcher 的调度 tiebreaker
- `workspace_kind` / `workspace_path`：三种 workspace 类型
- `max_runtime_seconds`：per-task 超时
- `current_run_id`：指向当前活跃的 run 行
- `scheduled_at`：延迟启动
- `block_recurrences`：unblock 循环计数器
- `consecutive_failures`：断路器计数器
- `workflow_template_id` / `current_step_key`：v2 预留

**`task_runs` 表**——每次尝试一行。字段包括 `profile`、`outcome`（`completed`/`blocked`/`timed_out`/`crashed`/`gave_up`/`protocol_violation`/`reclaimed`）、`summary`、`metadata`（JSON）、`started_at`、`ended_at`。这是重试的核心抽象——不是事后补救，是设计意图。

**`task_links` 表**——parent → child 依赖边。Dispatcher 在所有 parent 达到 `done` 之前不会把 child 从 `todo` 提升到 `ready`。

**`task_comments` 表**——agent 间协议。Agent 和人都可以追加评论；worker 在 spawn 时读到完整评论线程作为其上下文的一部分。

**`task_events` 表**——append-only 事件日志。每种状态转换产生一个事件行，带有可选的 `run_id` 以按尝试分组。`hermes kanban watch` 实时流式传输。

**`notify_subs` 表**——gateway 通知订阅。任务完成后自动推送到 Telegram/Discord/Slack 等通道。

### 4.2 调度器：60 秒心跳的协作引擎

Dispatcher 是一个长生命周期的循环，默认每 60 秒执行一次扫描。它**运行在 gateway 进程内部**（`kanban.dispatch_in_gateway: true`），不需要单独的守护进程。

每 tick 执行 4 项操作：

1. **回收 stale claims**：claim TTL 过期但任务还在 running——worker 可能挂了但 PID 还在。重置为 `ready`。
2. **回收崩溃 worker**：PID 没了但 TTL 还没过期。关闭 run 行，记录 `crashed` outcome，重置为 `ready`。
3. **提升 ready 任务**：检查 `todo` 任务——如果所有 parent 都是 `done`，提升到 `ready`。
4. **原子认领并 spawn**：从 `ready` 池中按 priority 选择任务，原子 CAS 认领（防止多 dispatcher 竞争），fork worker 进程。

Dispatcher 给 worker 子进程设置 `HERMES_KANBAN_TASK` 和 `HERMES_KANBAN_BOARD` 环境变量——worker 只能看到自己 board 上的任务。一个 dispatcher 扫所有 board。

防抖动：连续 spawn 失败达到 `kanban.failure_limit`（默认 2）后，dispatcher 自动 block 该任务并附上最后的错误信息——防止 profile 不存在或 workspace 挂载不了的任务无限重试。

### 4.3 并发策略：简单即正确

Kanban 的并发控制追求简单。没有分布式锁，没有 Raft 共识。策略就三步：

1. **WAL 模式**。SQLite WAL 允许多读单写，是本地并发的硬通货。
2. **BEGIN IMMEDIATE**。事务一开始就获取写锁，不等到第一个写操作。防止 deferred transaction 的写冲突。
3. **CAS 认领**。`UPDATE tasks SET status='running', current_run_id=X WHERE id=? AND status='ready'`——原子 compare-and-swap。如果两 dispatcher（不应该发生，但防御性编程）同时尝试认领同一任务，只有一个成功。

因为是单主机设计，SQLite 自身的序列化写者足够保证正确性。没有引入 etcd/ZooKeeper/Raft 的复杂性。

### 4.4 结构化交接：summary + metadata

Worker 完成任务时调用 `kanban_complete(summary=..., metadata={...})`。这不是一个日志消息——这是**下游 agent 的输入**。

- **summary**：可读摘要（1-3 句话），显示在 dashboard 和 CLI 的 `kanban runs` 输出中。
- **metadata**：机器可读的 JSON blob，包含 `changed_files`、`tests_run`、`decisions`、`residual_risk` 等。

当依赖链路上的下一个 worker 调用 `kanban_show()` 时，它会在 `worker_context` 中看到所有父任务的 summary 和 metadata。下游 agent 不需要重新阅读长设计文档——交接信息就在任务数据里。

推荐的 metadata 结构：

```json
{
  "changed_files": ["path/to/file.py"],
  "verification": ["pytest tests/ -q"],
  "dependencies": ["parent-task-id"],
  "blocked_reason": null,
  "retry_notes": "上次失败原因",
  "residual_risk": ["未测试的边界情况"]
}
```

这不是 schema 约束——是约定。但每个 worker 都留下足够的证据让下一个读者快速回答四个问题：
1. 改了什么？
2. 怎么验证的？
3. 失败了怎么恢复？
4. 还有什么风险刻意没处理？

---

## 五、失效处理：五种防线

Kanban 的失效处理是它跟 `delegate_task` 最本质的区别之一。不是一个 try/catch，是五层防线。

### 5.1 断路器（Circuit Breaker）

连续非成功尝试达到上限（默认 2，可分别通过 task 的 `max_retries`、dispatcher 的 `failure_limit` 或全局 `kanban.failure_limit` 覆盖）后，任务自动 block。不会无限重试烧 token。

断路器触发时发出 `gave_up` 事件，附带 `{failures, effective_limit, limit_source, error}` 信息，任务进入 blocked 状态等待人类检查。

### 5.2 Respawn Guard

Dispatcher 在重新 spawn 之前检查三项条件：

- **blocker_auth**：上次失败是配额/auth/429 错误——等 rate limit 窗口过去。
- **recent_success**：最近 1 小时内有完成的 run——等人类 review 后再跑。
- **active_pr**：最近的 comment 里有 GitHub PR URL——已有 worker 开了 PR。

这些 guard 不是由 LLM 决策的——是 DB 里的确定性检查。

### 5.3 Unblock 循环检测

如果一个任务被 block → unblock → re-block（同一原因）达到 `BLOCK_RECURRENCE_LIMIT`（默认 2）次，系统不再把它送回 `blocked` 状态——那样 cron 会继续 unblock 它——而是路由到 `triage`，要求人类做出决定。

这个计数器**刻意在 unblock 后存活**，只在成功 `complete` 时重置。这是为了防止 cron unblock → worker re-block 的死循环。是确定性 DB guard，不是 LLM 判断。

### 5.4 Heartbeat 降级

Worker 在执行长任务时应该至少每小时调用一次 `kanban_heartbeat`。如果任务运行时间超过 `kanban.dispatch_stale_timeout_seconds`（默认 4 小时）且最后 heartbeat 超过 1 小时，dispatcher 会：

1. SIGTERM 主机上的 worker（如果还在）
2. 任务回到 `ready` 重新 dispatch
3. **不计入失败计数器**——stale 是 dispatcher 侧缺席检测，不是 worker 故障

### 5.5 Protocol Violation 恢复

Worker 进程正常退出（exit code 0）但任务仍在 `running` 状态——通常是模型生成了文本回答后以 `finish_reason=stop` 退出，没调用 `kanban_complete` 或 `kanban_block`。

系统有两层防护：
- **Agent 侧**：退出前注入最多 2 次 nudges，提醒模型调 `kanban_complete` 或 `kanban_block`。
- **Dispatcher 侧**：nudge 耗尽或 worker 崩溃，dispatcher 给 protocol violation 有限重试（连续 3 次），然后 auto-block。

---

## 六、Kanban Swarm：多智能体协作拓扑

### 6.1 Swarm v1 拓扑

当前的 Swarm v1 实现一个简单的三段管道：

```
parallel workers → verifier → synthesizer
```

**Parallel workers** 是做实际工作的 agent。比如三个研究员并行查 ICP 融资数据。

**Verifier** 等所有 parallel workers 完成后启动（通过 `task_links` 依赖实现），检查输出质量。它不是"老板审批"——是自动化质量闸门。

**Synthesizer** 将验证过的输出合成为最终产物。

三者之间通过**共享黑板**通信：structured JSON comments 写在 task comment thread 里。这个设计刻意保持低技术含量——dashboard 能渲染它，notifier 能传递它，任何能读 comment 的组件都能消费它。

### 6.2 为什么 Swarm 不引入第二个调度器

这是最重要的架构决策之一。

很多 agent 框架会在现有系统上叠加一个"swarm orchestrator"——一个新的调度循环，管理 sub-agent 的生命周期。这会导致：
- 两个调度器的状态同步问题
- 故障域扩大（orchestrator 挂了全部完蛋）
- 可观测性分裂（orchestrator 的内部状态 vs board 的任务状态）

Kanban Swarm 走了一条不同的路：**它只是在 Kanban 任务板上写任务图**。Worker agent 用 `kanban_create` 创建子任务，用 `kanban_link` 建依赖关系，然后 dispatcher 照常调度。Swarm 是一个编排**模式**，不是一个运行时。

### 6.3 八种协作模式

从官方 spec 文档中提取的八种 canonical 协作模式：

1. **独立并行**：多个同质 worker 同时处理不同数据分片
2. **流水线**：A → B → C，每个阶段不同角色
3. **Map-Reduce**：map workers 并行处理 → reducer 聚合
4. **竞争式**：多个 worker 抢同一任务，先完成者胜
5. **审查-修订**：implementer → reviewer → implementer（循环）
6. **分治**：root task 递归分解子任务直到原子化
7. **人机协作**：agent 在关键决策点 block，等人类 unblock
8. **长期驻留**：digital twin profile 持续运行，每周/每日接收新任务

---

## 七、Triage 分解管道：从一句话到一张任务图

Triage 列是"想法停车场"。你把一句话扔进去——"研究 ICP 融资趋势"——系统自动把它展开成一张带依赖关系的子任务图。

这是由 `kanban_decompose.py` 驱动的，调用 `auxiliary.kanban_decomposer` 配置的 LLM。

分解器的输入：
- 原始 triage 任务的 title 和 body
- 你机器上安装的 profile roster（含 description）

分解器的输出：
- JSON 任务图：子任务列表，每个带 assignee 和 parents

关键安全机制：
- 分解器**永远不会产出 `assignee=None` 的子任务**。LLM 选了不存在的 profile？子任务路由到 `kanban.default_assignee`（如果没设置就 fallback 到 active default profile）。
- 原始 triage 任务成为所有子任务的 parent——所以它活着直到整个图完成，然后提升回 `ready` 让 orchestrator 判断是否还需要追加任务。

Auto 模式下，dispatcher 每 tick 最多分解 `kanban.auto_decompose_per_tick`（默认 3）个 triage 任务，防止批量灌入时瞬间烧完 LLM 预算。

---

## 八、竞品对比

Kanban 不是第一个多 agent 任务板。简单对比几个相关系统：

| 系统 | 存储 | 调度 | 交接 | 人类介入 | 故障恢复 |
|------|------|------|------|----------|----------|
| **Hermes Kanban** | SQLite（单文件，无依赖） | Gateway 内嵌 dispatcher，60s tick | Summary + metadata，下游 agent 直接消费 | Comment/unblock 任意时刻 | 断路器 + unblock 循环检测 + heartbeat + protocol violation |
| **Cline Kanban** | 基于文件系统（markdown） | 无自动调度 | 无结构——全靠 agent 读文本 | 手动 | 无 |
| **Paperclip** | In-memory agent graph | Centralized orchestrator | 无持久化——挂掉全丢 | 有限 | 依赖 orchestrator 存活 |
| **NanoClaw** | Filesystem + Redis | CLI 手动触发 | File-based | 手动 | 无自动化恢复 |
| **Google Gemini Enterprise** | Cloud-native | Managed | Vendor-locked | 有限 | 平台级 SLA |

Kanban 的设计哲学是**本地优先、零外部依赖、持久化不可协商**。它无意做云端的分布式任务系统——那是 Airflow/Temporal 的领域。它要做的是：在你的笔记本上，让 5 个 AI profile 像一个团队一样协作，不需要 internet、不需要 Kubernetes、不需要第三方服务。

---

## 九、实践案例

### 案例 1：研究管线

**场景**：调研 ICP 融资趋势，产出一篇分析文章。

**设置**：
- `researcher-a`：北美市场
- `researcher-b`：欧洲市场
- `writer`：合成分析文章

**流程**：
1. 在 triage 列创建 "ICP funding landscape research"
2. Decomposer 自动产出：researcher-a → 北美，researcher-b → 欧洲，writer 依赖两者
3. 两个 researcher 并行工作，各自完成后交接 summary + metadata
4. Writer 启动时看到两个父任务的 handoff，无需重新阅读全文
5. 如果 writer 不满意某个研究结果，可以 `kanban_comment` 到对应 researcher 的任务要求补充，然后 block 自己的任务等更新

### 案例 2：代码审查管道

**场景**：实现新 feature，带 mandatory code review。

**设置**：
- `backend-dev`
- `reviewer`

**流程**：
1. 创建 "Implement auth middleware" → `backend-dev`
2. Backend-dev 完成后，reviewer 任务自动提升到 ready（通过 `task_links`）
3. Reviewer 审查——如果通过，complete；如果不通过，block reviewer 任务，comment 到 implementer 任务要求修改
4. Implementer 修改后重新 complete，reviewer 任务 unblock → ready → 重新 dispatch

### 案例 3：Fleet 运营

**场景**：一个 `translator` profile 管理 12 个翻译任务，一个 `transcriber` profile 处理 5 个音频转录。

**设置**：两个 profile 各有独立的任务池，通过 `tenant` tag 分区。

**流程**：
1. 批量创建任务，按 assignee 自动分配
2. Gateway + dispatcher 让两个 worker 并行拉取各自任务池
3. Dashboard 的 "Lanes by profile" 视图让你一眼看到每个 profile 当前在做什么
4. 一个 worker 完成当前任务后，dispatcher 自动分配下一个 ready 任务

不需要编写任何调度代码。整个系统就是 SQLite 里的状态机加上 gateway 里的 dispatcher loop。

---

## 十、最佳实践

### 10.1 Profile 设计

- 每个 profile 配一个**描述**（`hermes profile describe <name> --text "..."`）。Triage 分解器用它做路由决策。
- Orchestrator profile 的 toolsets 应该只包含 `kanban`、`gateway`、`memory`——不该有 `terminal` 或 `file`。让 orchestrator 只能编排，不能执行。这不是信任问题，是防止 scope creep。

### 10.2 任务拆分粒度

- 一个任务应该是一个 profile 可以在单次 run 内完成的工作单元。
- 如果任务需要超过 1 小时，确保 worker 每小时至少调一次 `kanban_heartbeat`。
- 如果任务本身的复杂度跨多个角色——拆。用 `kanban_create` + `parents` 建依赖链。

### 10.3 交接信息密度

- `summary` 写给人看（dashboard），`metadata` 写给下游 agent 看。
- Summary 保持 1-3 句——不要贴整个 terminal log。
- Metadata 必须包含验证证据（tests_run、verification 命令），不只声明"完成了"。
- 不要往 metadata 里放 token、密钥、OAuth 材料。

### 10.4 故障模式思维

- 总假设 worker **会挂**。不要指望一次 run 就成功。
- 在设计任务时就想好：如果这次 run 的 worker 挂了，下一个接手 worker 能从 `worker_context` 中获取足够信息继续吗？
- 使用 typed block（`dependency` / `needs_input` / `capability` / `transient`）而不是 generic block。Typed block 让 unblock 循环检测能正确区分"等依赖"和"真的需要人"。

### 10.5 Board 设计

- 一个 project/repo/domain = 一个 board。硬隔离，不跨板建依赖。
- 用 `tenant` 做板内的软分区——比如一个 `content-ops` board 下，`tenant=blog` 和 `tenant=social` 分区。
- 定期归档 done 任务：`hermes kanban archive <id>`。

### 10.6 Gateway 运营

```yaml
# config.yaml
kanban:
  dispatch_in_gateway: true        # dispatcher 跑在 gateway 里，不需要额外进程
  dispatch_interval_seconds: 60    # 扫描间隔
  auto_decompose: true             # triage 自动分解
  auto_decompose_per_tick: 3       # 每 tick 最多分解 3 个，防止 LLM 预算爆炸
  orchestrator_profile: "orchestrator"  # 分解后的根任务 assignee
  failure_limit: 3                 # 连续失败上限（默认 2）
  dispatch_stale_timeout_seconds: 14400  # 4 小时无 heartbeat 回收
```

启动后一个命令即可：

```bash
hermes gateway start
```

Gateway 启动后，dispatcher 自动开始在每个 tick 调度 ready 任务。不需要 systemd timer、不需要 cron、不需要 supervisor——gateway 就是你的多 agent 运行时。

---

## 结语

Kanban 解决了一个具体问题：**如何让多个 AI profile 在一台机器上可靠地协作，同时保持对人类完全透明**。它不做分布式集群——那是 Kubernetes 的活。它不做实时——那是消息队列的活。它做的是：把你的 AI 团队的状态持久化在一个你可以随时打开看的 SQLite 文件里，让每个 worker 每次 run 都有完整的审计追踪，让挂掉的 worker 可以被回收重试而不是悄无声息地死掉。

如果你在本地跑了 3 个以上的 AI agent，或者你有一个定期运行的 AI 任务需要人工偶尔插手的场景——Kanban 是目前最务实的答案。

---

*本文基于 Hermes Kanban v1 的源码分析（`kanban_db.py`、`kanban_swarm.py`、`kanban.py`、`kanban_decompose.py`）和官方文档撰写。*
