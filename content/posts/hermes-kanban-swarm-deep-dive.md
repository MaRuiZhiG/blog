---
title: "Hermes Agent Kanban Swarm 深度解析：多 Agent 协作的持久化编排引擎"
date: 2026-07-04T22:20:00+08:00
tags: ["hermes", "kanban", "swarm", "multi-agent", "orchestration", "architecture", "deep-dive"]
author: "MaRuiZhi"
slug: "hermes-kanban-swarm-deep-dive"
description: "深度解析 Hermes Agent 的 Kanban Swarm 编排引擎：从 delegate_task 到持久化任务队列的架构演进、DAG 拓扑、Dispatcher 调度、容错与可恢复性设计。"
draft: false
---

# Hermes Agent Kanban Swarm 深度解析：多 Agent 协作的持久化编排引擎

> 面向 AI 开发者的架构级解读。本文假定你已理解 agent 的基本概念（tool-calling、system prompt、context window），重点拆解 Kanban Swarm 如何将"多 agent 协作"从脆弱的进程内调用升级为可恢复、可审计、可人工介入的生产级原语。

---

## 1. 一句话定位

`hermes kanban swarm` 不是一个 "agent 框架"——它是**持久化任务队列 + 并行拓扑模板的语法糖**。它的底层是 SQLite 中的任务状态机，上层是一个固定的 DAG 模板：**N 个并行 worker → 1 个 verifier（门禁审查）→ 1 个 synthesizer（最终合成）**。每一个节点是**一个独立 OS 进程**，身份由 Hermes 的 profile 系统定义，拥有独立的 memory、context 和工具集。

---

## 2. 为什么需要 Kanban？`delegate_task` 不够吗？

Hermes 本身已有 `delegate_task` —— 父 agent 可以 spawn 子 agent，子 agent 完成后返回结果。这看起来像多 agent 协作，但本质上是一个 **RPC 调用**：

| 维度 | `delegate_task` | Kanban (含 Swarm) |
|------|----------------|-------------------|
| **模型** | fork → join (函数调用) | 消息队列 + 状态机 |
| **父 agent 行为** | 阻塞等待子 agent 返回 | 创建 task 后即返回，dispatcher 异步调度 |
| **子 agent 身份** | 匿名子进程 | 具名 profile，拥有持久化 memory |
| **可恢复性** | 失败即失败 | task 可 block → unblock → 重试；crash 后 dispatcher 回收 |
| **人工介入** | 不支持 | 任何时刻 comment/unblock |
| **审计** | context 压缩后丢失 | SQLite 中永久保留每条 handoff 记录 |
| **每 task 的 agent 数** | 1 | N 个（重试、审查、跟进） |
| **批量能力** | 单次调用 spawn 1 个子 agent | 无限制（fleet farming） |
| **并发上限** | 支持 `tasks` 数组最多 3 并发 | 受 `kanban.max_concurrent` 限制 |

用一句话区分：**`delegate_task` 是函数调用；Kanban 是工作队列。**Swarm 在这个队列之上叠加了固定的并行/串行拓扑。

> **补充**：`delegate_task` 支持 `tasks` 数组模式，可一次性派发最多 3 个并行子 agent（受 `delegation.max_concurrent_children` 控制）。但每个子 agent 仍是匿名的、不可恢复的，一旦失败就丢失。

---

## 3. Swarm 的拓扑结构

`kanban swarm` 创建的 DAG 是硬编码的（源码 `kanban_swarm.py:6-9`）：

```
Root (立即标记 done，仅作共享黑板/审计锚点)
  ├─ Worker 1 ──┐
  ├─ Worker 2 ──┤  三者并行执行
  ├─ Worker 3 ──┘
  └─ Verifier       ← 等所有 worker done，审查输出
       └─ Synthesizer ← 等 verifier gate=pass，合成最终产物
```

### 3.1 每个节点是什么？

每个节点是 `kanban.db` 中的一行 task，不是一个线程或协程。**Swarm 不"运行"任何东西**——它只是创建 task 和它们的依赖关系（`task_links` 表中的 parent→child 行）。真正执行由 **Dispatcher** 负责。

### 3.2 依赖关系如何驱动执行

```
task_links 表:
  verifier.parent = [worker1, worker2, worker3]  → verifier 初始状态: todo
  synthesizer.parent = [verifier]                → synthesizer 初始状态: todo
  worker[1-3].parent = [root]                    → 三个 worker 初始状态: ready (因为 root 已经是 done)
```

Dispatcher 每 60 秒扫描一次：

1. 发现 worker1/2/3 都是 `ready` → 认领（原子 CAS 操作）→ spawn 三个独立 OS 进程
2. 三个 worker 并行运行，各自完成后调用 `kanban_complete()` → 状态变为 `done`
3. Dispatcher 发现 verifier 的所有父 task 都是 `done` → 自动将 verifier 从 `todo` 晋升为 `ready`
4. Verifier 被 spawn，完成审查后标记 `done`
5. Synthesizer 晋升 → spawn → 合成 → done

这就是一个完整的、异步推进的 **DAG 执行引擎**。无需中心化调度器理解"任务语义"——它只回应状态变化。

---

## 4. 任务的完整生命周期

每个 task 经历 7 个状态，这是一个严格的状态机：

```
triage → todo → ready → running → done
                    ↘ blocked
                    ↘ archived
```

### 关键状态说明

- **triage**：草稿状态。只有标题，可能只是一句话的想法。Dispatcher **不会**自动执行 triage 任务。
- **todo**：已有完整规格，但在等待父 task 完成。
- **ready**：所有依赖已满足，等待下一次 dispatcher tick 时被认领。
- **running**：worker 进程中。每 60 秒发一次 heartbeat，超时或 crash 后 dispatcher 自动回收。
- **blocked**：worker 主动标记（如需要人工决策），或 `kanban.failure_limit`（`config.yaml` 中配置，默认 2）次连续失败后自动 block（断路器模式）。
- **done**：完成。**scratch workspace 被删除**（重要！）—— 如果要保留产物，必须用 `--workspace dir:<path>` 或 `worktree`。

---

## 5. Dispatcher：调度引擎的核心

Dispatcher 不是一个 Kubernetes scheduler。它是一个**运行在 gateway 进程内的 60 秒循环**，代码在 `gateway/kanban_watchers.py`。每个 tick 做的事情：

```bash
for each tick:
    step 0: 检查 scheduled_at 到期任务 → 从 todo 自动晋升为 ready
    step 1: 回收超时/崩溃的 worker（PID 不存在 或 heartbeat 超时）
    step 2: 提升 ready 任务——扫描所有 todo 任务，父 task 全部 done 的 → 设为 ready
    step 3: 认领并 spawn——原子 CAS 更新 status (ready→running) + claim_token
    step 4: 如果 spawn 失败超过 kanban.failure_limit（默认 2）→ 自动 block（防抖/断路器）
```

### 5.1 Worker spawn 的真相

这不是 Python `multiprocessing`。Dispatcher 调用 `hermes agent run --profile <name> ...` 启动一个**全新的 OS 进程**：

- 继承 profile 的所有配置（SOUL.md、toolsets、memory provider）
- 自动注入 `kanban_*` 工具集（`kanban_show`, `kanban_complete`, `kanban_block`, `kanban_heartbeat`, `kanban_comment`, `kanban_create`, `kanban_link`, `kanban_list`, `kanban_unblock`）。注意 `kanban_list` 和 `kanban_unblock` 仅 orchestrator profile 可用——普通 worker 只能操作自己的 task。
- 环境变量 `HERMES_KANBAN_TASK=<task_id>` 告诉 worker 它的任务 ID
- 每个 worker 有独立的 `cwd`（scratch workspace / worktree / shared dir）

### 5.2 Heartbeat 与崩溃恢复

Worker 运行时通过 `kanban_heartbeat(note="...")` 定期发送心跳（默认每 60 秒，由 `kanban.heartbeat_interval_seconds` 配置）。Dispatcher 检测超时的手段：

- 检查 claim 的 TTL 是否过期
- 检查 PID 是否还活着

如果 worker 崩溃，Dispatcher 关闭当前 run 记录（outcome=`crashed`），将 task 重置为 `ready`，下一个 tick 重新 spawn。这意味着 **worker 崩溃不会丢失工作进度**——comment 历史、工作目录状态（如果用的是持久化 workspace）都保留。

---

## 6. Worker 间的通信协议：Comment + Handoff

Swarm 拓扑中，worker 之间**不直接通信**。它们通过 Kanban board 上的两个机制交换信息：

### 6.1 Comment（评论）

任何 profile 或人类都可以通过 `kanban_comment(task_id, text)` 对任何 task 追加评论。Worker 被 spawn 时会读取该 task 的完整评论线程作为 context。

### 6.2 Handoff（交接）

Worker 完成任务时调用 `kanban_complete(summary="...", metadata={...})`。这是**结构化的交接信**，包含：

- `summary`：自然语言总结（例如 `"users 表 id/email/pw_hash, sessions 表 id/user_id/jti/expires_at"`）
- `metadata`：结构化数据（例如 `{"changed_files": [...], "decisions": [...]}`）

下游 worker（verifier → synthesizer）通过 `kanban_show()` 读取上游的 handoff，获得前面的所有决策上下文，而不需要解析长文本文档。

---

## 7. Shared Blackboard（共享黑板）

Root task 的特殊之处：它在 swarm 创建时就被标记为 `done`，不执行任何工作。但它是所有 worker 的共同父节点，因此：

- **任何人都可以在 root task 上 comment**，作为全局共享信息
- Worker 通过 `kanban_show(root_id)` 读取 root 的评论 → 获得全局 context
- Root 是审计锚点——所有 worker 的依赖关系都回溯到它

这实现了一种**黑板架构（blackboard pattern）**：多个专家 agent 读写共享数据结构，协调工作而不需要点对点通信。

---

## 8. 与 Decomposer（自动分解）的对比

Kanban 提供两种构建任务图的方式：

| 方式 | 命令 | 适用场景 |
|------|------|---------|
| **Swarm**（手动拓扑） | `hermes kanban swarm "目标" --worker ...` | 你清楚怎么拆：3 个活分头干，然后审查+合成 |
| **Decomposer**（LLM 分解） | `hermes kanban create "目标" --triage` + `kanban decompose <id>` | 不知道具体怎么拆："帮我分一下" |

Decomposer 的工作流程：

1. 辅助 LLM（`auxiliary.kanban_decomposer` 配置）读取所有 profile 的 name + description
2. LLM 返回 JSON 子任务图，每个子任务含 `{title, body, assignee, parents}`
3. LLM 按 profile description 语义匹配 assignee，不匹配的 fallback 到 `default_assignee`
4. 根任务分配给 `orchestrator_profile`（验收者，子任务全部完成后被唤醒）

Swarm 是语法糖——等效于手动 `kanban create` + `kanban link` 但一行搞定。Decomposer 是自动化——LLM 决策分工。

---

## 9. 技术实现：从概念到代码

### 9.1 数据层（`kanban_db.py`）

所有操作最终落到 SQLite。默认 board 的 DB 路径为 `~/.hermes/kanban.db`，自定义 board 为 `~/.hermes/kanban/boards/<board>/kanban.db`。核心表结构：

```sql
-- tasks 表
CREATE TABLE tasks (
    id INTEGER PRIMARY KEY,
    title TEXT NOT NULL,
    body TEXT,
    assignee TEXT,        -- profile 名称
    status TEXT,          -- triage|todo|ready|running|blocked|done|archived
    workspace_type TEXT,  -- scratch|dir|worktree
    workspace_path TEXT,
    claim_token TEXT,     -- 原子认领的 CAS token
    claimed_at INTEGER,
    heartbeat_at INTEGER,
    tenant TEXT
);

-- task_links 表（依赖关系）
CREATE TABLE task_links (
    parent_id INTEGER,
    child_id INTEGER
);

-- task_runs 表（每次执行记录）
CREATE TABLE task_runs (
    id INTEGER PRIMARY KEY,
    task_id INTEGER,
    profile TEXT,
    outcome TEXT,         -- active|completed|gave_up|timed_out|crashed
    summary TEXT,
    metadata TEXT,        -- JSON
    started_at INTEGER,
    ended_at INTEGER
);
```

### 9.2 Swarm 创建过程（`kanban_swarm.py`）

核心逻辑（简化自源码 `create_swarm()` 函数）：

```
Step 1: 创建 root task（直接标记 done，成为共享黑板）
root_id = kanban_db.create_task(title=goal, assignee=orchestrator, status="done")

Step 2: 创建 N 个 worker task（status=ready，因为 root 已 done）
for each worker_spec in workers:
    wid = kanban_db.create_task(title=worker_spec.title, assignee=worker_spec.profile, status="ready")
    kanban_db.add_link(root_id, wid)   # parent=root
    worker_ids.append(wid)

Step 3: 创建 verifier（status=todo，依赖所有 worker）
vid = kanban_db.create_task(title="Verify: " + goal, assignee=verifier, status="todo")
for wid in worker_ids:
    kanban_db.add_link(wid, vid)       # parent=每个worker

Step 4: 创建 synthesizer（status=todo，依赖 verifier）
sid = kanban_db.create_task(title="Synthesize: " + goal, assignee=synthesizer, status="todo")
kanban_db.add_link(vid, sid)           # parent=verifier
```

### 9.3 Worker Context 注入

每个 worker 收到特殊的 context 注入（`kanban_swarm.py` 的 `_swarm_context()` 函数）：

```
You are worker N of M in a Kanban Swarm.
- Goal: <root task body>
- Your role: <worker title>
- Shared blackboard: task <root_id> (read comments there for global coordination)
- When done: call kanban_complete() with summary + metadata
```

---

## 10. 实际命令示例

```bash
# 创建一个技术调研 Swarm
hermes kanban swarm "调研三种向量数据库并写对比报告" \
  --worker "researcher:调研 Milvus" \
  --worker "researcher:调研 Qdrant" \
  --worker "researcher:调研 Weaviate" \
  --verifier reviewer \
  --synthesizer writer

# 启动 gateway（dispatcher 在里面）
hermes gateway start

# 查看进度
hermes kanban list
hermes kanban show <task_id>

# 如果 verifier 发现 worker 的调研不充分，可以 blocked 并 comment
# dispatcher 不会 auto-reclaim blocked task
# 人类手动 kanban unblock <id> 后重新进入 ready
```

---

## 11. 设计哲学：为什么不是"Agent Swarm"

注意这里的 "Swarm" 不是常见的 agent-swarm 概念（如 OpenAI Swarm、AutoGen）。Kanban Swarm 不涉及：

- 不涉及 Agent 之间的实时消息传递
- 不涉及动态路由或自适应拓扑
- 不涉及进程内 agent 对象的相互调用

而是：

- DAG 依赖驱动的任务状态机
- 完全异步——创建后就 fire-and-forget
- 持久化 first——每个 handoff 是 SQLite 一行，不怕 crash
- 人工可介入——任何时刻可以 comment、block、unblock
- 可审计——谁在什么时候做了什么，永久记录

这不是"智能体集群"的 romantic vision，而是**生产级的异步任务编排**。Hermes 的赌注是：对于真实世界的 agent 工作流，可靠性 > 灵活性。

---

## 12. 局限与适用边界

### 当前局限

- **拓扑是固定的**：Swarm v1 只支持 `worker → verifier → synthesizer`，不支持自定义 DAG（但可以手动 `kanban create` + `kanban link` 构建任意拓扑）
- **单机运行**：Kanban 的威胁模型是"信任的单用户主机"，没有分布式调度。Dispatcher 跑在 gateway 里，worker 跑在本机
- **没有动态路由**：不会根据 worker 的中间结果动态创建新 task（但 orchestrator 可以在子任务完成后被唤醒并创建新任务）
- **LLM 驱动的 decomposer 依赖辅助 LLM**：需要配置 `auxiliary.kanban_decomposer`

### 适合的场景

- 研究审阅流水线（多个研究员并行 → 审查员 → 撰稿人）
- CI 风格的编码流水线（实现 → review → 测试 → PR）
- 定期运营任务（cron + kanban 组合）
- 需要人工在环的复杂工作流
- 多人/多 profile 的协作场景

### 不适合的场景

- 需要毫秒级响应的实时任务
- 需要 agent 间密集交互的对话式协作
- 分布式/多机器部署（当前设计是单机的）

---

## 13. 核心要点总结

1. **Swarm 是 DAG 模板，不是运行引擎**——它创建 task 和依赖关系，真正的调度由 Dispatcher 负责
2. **每个 worker 是独立 OS 进程**——不是 goroutine/thread/协程，拥有独立 memory 和 identity
3. **持久化是核心竞争力**——SQLite 中的每个 handoff 让你可以在任何时刻恢复、审计、人工介入
4. **Heartbeat + 崩溃回收 = 自愈**——worker crash 后 dispatcher 自动重置 task 并重新 spawn
5. **仅 1 条命令构建完整流水线**——`--worker` × N + `--verifier` + `--synthesizer`，不需要写任何编排代码
6. **黑板模式实现 agent 间通信**——不是直接消息传递，而是通过 comment + handoff 读写共享 board

---

*Hermes Kanban Swarm 代表了 AI agent 工程化的一个关键趋势：从"智能对话"到"可靠的异步工作流"。它不是最智能的，但是最耐操的。*

---

## 参考

- [Kanban 官方文档](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban)
- [Kanban 教程](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban-tutorial)
- 源码：`hermes_cli/kanban_swarm.py`（Swarm 拓扑创建）
- 源码：`gateway/kanban_watchers.py`（Dispatcher 循环）
- 源码：`hermes_cli/kanban_db.py`（SQLite 操作层）
- 源码：`hermes_cli/kanban_decompose.py`（LLM 自动分解）
- 设计规格：repo 中的 `docs/hermes-kanban-v1-spec.pdf`
