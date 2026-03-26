# s07 - Task System

> **核心问题**: 如何让 Agent 的目标比一次对话更持久？
> **参考文档**: `docs/zh/s07-task-system.md`
> **参考实现**: `agents/s07_task_system.py`
> **状态**: ✅ 已完成

---

## 一、知识总结

### 本节核心概念

**一句话本质**：把 Agent 的任务从「活在 messages 里的扁平清单」升级为「持久化到磁盘的有向无环图（DAG）」，让目标跨越会话边界存活。

### 关键设计决策表

| 问题 | 决策 | 原因 |
|------|------|------|
| 任务存哪里？ | 磁盘 JSON 文件（每个任务一个文件） | 读磁盘是天然的 tool call，不需要额外基础设施 |
| 用什么结构表示依赖？ | DAG，双向存边（`blockedBy` + `blocks`） | 单向只存前置依赖，无法从单个文件看出下游影响 |
| 何时解锁后续任务？ | 仅在 `completed` 时触发，`in_progress` 不触发 | 产出物未就绪前解锁，后续任务会拿到半成品 |
| 如何防止多 Agent 重复认领？ | `owner` 字段记录执行者 ID | 认领时写入 owner，其他 Agent 跳过非空 owner 的任务 |
| 如何扩展工具？ | 往 dispatch map 加 4 个条目，核心循环不改 | 开闭原则：对扩展开放，对修改关闭 |

### 任务数据结构

```python
task = {
    "id": 1,
    "subject": "任务目标描述",   # 任务是什么
    "status": "pending",         # pending → in_progress → completed
    "blockedBy": [],             # 前置依赖（等谁完成）
    "blocks": [],                # 后置依赖（完成后解锁谁）
    "owner": ""                  # 谁在执行（多 Agent 防冲突）
}
```

### 关键代码片段与对应原理

**依赖解除机制**——任务完成时扫描所有文件，把自己的 ID 从他人的 `blockedBy` 中移除：

```python
def _clear_dependency(self, completed_id):
    for f in self.dir.glob("task_*.json"):
        task = json.loads(f.read_text())
        if completed_id in task.get("blockedBy", []):
            task["blockedBy"].remove(completed_id)
            self._save(task)
```

注意：没有用 `blocks` 字段来找后续任务，而是全量扫描。`blocks` 是为了可读性，让单个文件就能看到完整上下游。

**可执行任务判断条件**：`status == "pending"` 且 `blockedBy == []`

### 架构变更对比

| 组件 | s06 之前 | s07 之后 |
|------|----------|----------|
| 规划模型 | 扁平清单（仅内存） | DAG 任务图（磁盘持久化） |
| 依赖关系 | 无 | `blockedBy` + `blocks` |
| 状态 | 做完 / 没做完 | `pending` → `in_progress` → `completed` |
| 持久化 | 压缩/重启后丢失 | 压缩和重启后存活 |
| 工具数量 | 4 | 8（新增 task_create/update/list/get） |

### s03 TodoManager vs s07 TaskManager

| | TodoManager (s03) | TaskManager (s07) |
|--|-------------------|--------------------|
| 存储位置 | 内存 / messages | 磁盘 JSON 文件 |
| 结构 | 扁平清单 | DAG（有依赖关系） |
| 生命周期 | 会话结束消失 | 跨会话持久存活 |
| 用途 | 单次会话内快速清单 | 多步骤、跨会话长任务 |

### 可泛化原则

- **状态机 + 持久化**：任何需要跨进程/跨会话续跑的系统，都需要把状态外化到可靠存储（数据库、文件、KV）
- **DAG 调度**：CI/CD（GitHub Actions）、数据管道（Airflow）、构建系统（Gradle）都用同一个模式——有向无环图 + 依赖解除
- **协调中心模式**：多执行者共享一份状态存储，靠状态字段（status/owner）协调，而非点对点通信

---

## 二、学习过程反思

### 答对的地方

- 正确判断现有 Agent 无法处理跨会话任务，指出 messages 不持久是根本原因
- 正确推导出需要 task_id、description、status 三个基础字段
- 正确分析出 B（磁盘文件）优于 A（messages）和 C（数据库）
- 正确理解 `in_progress` 不触发依赖解除的原因（产出物未就绪）
- 正确识别 `owner` 字段用于多 Agent 防止重复认领

### 犯的错误或思维漏洞

- **TodoManager vs TaskManager 的本质区别**：最初回答"一个 task 对应多个 todo"的层级关系，这是错误的。正确理解是：两者都是追踪任务的工具，核心区别是**持久化**——TodoManager 活在内存里，TaskManager 存在磁盘上。

- **`blocks` 字段的作用**：猜测是用于"完成后触发后续任务"，但实际实现是全量扫描 `blockedBy`，而非用 `blocks` 主动通知。`blocks` 的真正作用是可读性（打开单个文件就能看到完整上下游），不是执行触发。

### 思考不够清晰的地方

- dispatch map 对应的设计原则（开闭原则）叫法需要巩固，概念本身理解到位但名称记忆模糊

---

## 三、教学引导记录

### 产生顿悟的类比/问题

- "读磁盘文件对 Agent 来说本身就是 tool call，天然契合已有工具体系" ——这个角度让存磁盘的选择从"够用就好"变成了"架构一致性的最优解"

- 三个状态（pending/in_progress/completed）的语义拆解：pending=等条件，in_progress=执行中产出物未就绪，completed=产出物已就绪可解锁——让状态机设计从"随意命名"变成了有业务语义的决策

### 需要后续关注的跨课程线索

- `owner` 字段在 s09 多 Agent 团队中如何实际使用（任务认领的竞争问题）
- s08 后台任务如何与任务图的 `in_progress` 状态配合
- s12 Worktree 隔离中，多 Agent 同时写任务图文件时如何处理并发冲突

---

*学习日期：2026-03-26*
*提交 commit：待提交*
