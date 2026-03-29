# s12 - Worktree Task Isolation

> **核心问题**: 多个 Agent 同时修改代码怎样不冲突？
> **参考文档**: `docs/zh/s12-worktree-task-isolation.md`
> **参考实现**: `agents/s12_worktree_task_isolation.py`
> **状态**: ✅ 已完成

---

## 一、知识总结

### 本节核心概念

**一句话本质**：给每个任务分配独立的 git worktree 目录，通过控制平面（.tasks/）和执行平面（.worktrees/）分离，并用事件流记录生命周期，解决多 Agent 并发修改代码的冲突问题。

### 关键设计决策表

| 问题 | 决策 | 原因 |
|------|------|------|
| 如何隔离多 Agent 的文件修改？ | git worktree：每个任务独立目录 | 同一仓库的不同分支检出到不同目录，互不干扰 |
| 为什么不直接用 git 分支？ | 分支共享工作目录 | checkout 会修改共享目录，影响其他 Agent |
| 为什么创建 worktree 自动推进任务状态？ | 两步合并为一，最小化不一致窗口 | 分两步中间崩溃会留下孤立 worktree + pending 任务 |
| worktree 名唯一性怎么保证？ | 文件系统天然唯一 | 第二个同名 worktree 创建时 git 报错，充当软锁 |
| 为什么需要 events.jsonl？ | 时间线 vs 快照互补 | .tasks/ 是当前状态，events.jsonl 是变化历史，崩溃后能定位断点 |
| 为什么 remove + complete_task 合并？ | 原子性，避免僵尸任务 | 分步执行中间崩溃会留下 in_progress 的任务永远无人认领 |

### 控制平面 / 执行平面分离

```
控制平面 (.tasks/)              执行平面 (.worktrees/)
+------------------+            +------------------------+
| task_1.json      |            | auth-refactor/         |
|   status: in_progress <-----> |   branch: wt/auth      |
|   worktree: "auth-refactor"   |   task_id: 1           |
+------------------+            +------------------------+
| task_2.json      |            | ui-login/              |
|   status: pending    <------> |   branch: wt/ui-login  |
|   worktree: "ui-login"        |   task_id: 2           |
+------------------+            +------------------------+
                                | index.json（注册表）    |
                                | events.jsonl（事件流）  |
```

**类比**：Kubernetes（control plane 管 desired state，data plane 执行）、网络交换机（路由表 vs 转发芯片）、GitHub Actions（YAML 定义 vs Runner 执行）

### 快照 vs 时间线

| | .tasks/ | events.jsonl |
|--|---------|-------------|
| 记录内容 | 当前状态 | 变化历史 |
| 类比 | 银行账户余额 | 银行流水 |
| 用途 | 快速重建现场 | 崩溃后定位断点 |
| 能否互推 | 流水可以重算余额 | 余额不能还原流水 |

### 关键代码片段

**双向绑定：创建 worktree 自动推进任务状态**
```python
def bind_worktree(self, task_id, worktree):
    task = self._load(task_id)
    task["worktree"] = worktree
    if task["status"] == "pending":
        task["status"] = "in_progress"   # 绑定时自动推进
    self._save(task)
```

**原子收尾：删目录 + 完成任务 + 发事件三合一**
```python
def remove(self, name, force=False, complete_task=False):
    self._run_git(["worktree", "remove", wt["path"]])
    if complete_task and wt.get("task_id") is not None:
        self.tasks.update(wt["task_id"], status="completed")
        self.tasks.unbind_worktree(wt["task_id"])
        self.events.emit("task.completed", ...)
```

### 架构变更对比

| 组件 | s11 之前 | s12 之后 |
|------|----------|----------|
| 执行范围 | 共享目录 | 每个任务独立目录 |
| 协调 | 任务板（owner/status） | 任务板 + worktree 显式绑定 |
| 可恢复性 | 仅任务状态 | 任务状态 + worktree 索引 |
| 收尾 | 任务完成 | 任务完成 + 显式 keep/remove |
| 生命周期可见性 | 隐式 | events.jsonl 显式事件流 |

### 可泛化原则

- **控制平面/执行平面分离**：Kubernetes、网络设备、CI/CD——分离"想要什么"和"怎么做"，控制平面可以独立重建
- **快照 + 时间线互补**：数据库（状态表 + binlog）、版本控制（当前文件 + commit history）——两者配合实现完整的可观测性和可恢复性
- **原子操作减少中间状态**：数据库事务、文件系统 rename——把多步操作合并，最小化不一致窗口
- **物理隔离 > 逻辑隔离**：当并发冲突代价高时，用独立目录/进程/容器比用锁更简单可靠

---

## 二、学习过程反思

### 答对的地方

- 正确推导出用 git 分支隔离的思路
- 正确理解 worktree 是"同一仓库多个工作目录"
- 正确理解 events.jsonl 的"时间线"作用（快照无法还原历史）
- 正确理解原子收尾的价值（分步执行中间崩溃留僵尸任务）

### 犯的错误或思维漏洞

- **创建 worktree 自动推进任务状态的原因**：最初回答"保证在 git 管理范围内"，方向有道理但不够精确。核心原因是：两步分开执行中间崩溃会导致 worktree 存在但任务仍是 pending，被其他 Agent 重复认领。

### 思考不够清晰的地方

- 控制平面/执行平面分离的概念在工程领域有很广泛的应用（K8s、网络设备、CI/CD），需要建立更系统的认知框架

---

## 三、教学引导记录

### 产生顿悟的类比/问题

- "三个 Agent 同时 git checkout 会发生什么？"——让"为什么需要独立工作目录"有了具体的失败场景，而不是抽象的"会冲突"

- 控制平面/执行平面的多个类比（K8s、交换机、GitHub Actions）——让这个架构模式从 s12 的一个设计选择变成了可泛化的工程原则

### 需要后续关注的跨课程线索

- 生产级 worktree 管理：多个 worktree 同时 push 到 remote 时的合并策略
- 事件流的持久化保证：events.jsonl 本身如果在写入时崩溃怎么办（partial write）

---

## 全课程总结：从 s01 到 s12

这 12 节课做了同一件事：把一个"只能在一次对话里思考"的 LLM，变成一个"能可靠完成真实世界复杂任务"的 Agent。

三个维度的突破：
- **时间**：s07/s08 让任务比一次对话更长（持久化 + 异步）
- **空间**：s04/s05/s06 让任务比一个上下文窗口更大（隔离 + 知识 + 压缩）
- **规模**：s09-s12 让任务比一个 Agent 更复杂（团队 + 协议 + 自治 + 隔离）

架构演进本质是不断外化状态：
```
内存 messages → 磁盘 .tasks/ → 磁盘 .worktrees/ → git history
```
越往后，状态越持久、越可靠、越能在崩溃后重建。

---

*学习日期：2026-03-29*
*提交 commit：待提交*
