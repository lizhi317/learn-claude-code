# Agent Harness 学习进度总览

> 目标：系统掌握 Agent Harness 工程的完整知识体系，从 30 行的 while 循环到完整的多 Agent 协作系统。

## 进度追踪表

| # | 课程 | 核心问题 | 状态 | 笔记 | 完成日期 |
|---|------|----------|------|------|----------|
| **第一阶段：循环** |
| 1 | s01 - Agent Loop | AI 模型如何从"只能说话"变成"能做事"？ | ✅ 已完成 | [→](phase1-the-loop/s01-agent-loop.md) | 2026-03-21 |
| 2 | s02 - Tool Use | 怎样让 Agent 拥有更多能力而不修改核心循环？ | ✅ 已完成 | [→](phase1-the-loop/s02-tool-use.md) | 2026-03-21 |
| **第二阶段：规划与知识** |
| 3 | s03 - TodoWrite | 为什么 Agent 做长任务会"跑偏"？ | ✅ 已完成 | [→](phase2-planning-knowledge/s03-todo-write.md) | 2026-03-26 |
| 4 | s04 - Subagent | 为什么需要"用完即扔"的子智能体？ | ✅ 已完成 | [→](phase2-planning-knowledge/s04-subagent.md) | 2026-03-26 |
| 5 | s05 - Skill Loading | 怎样让 Agent 拥有领域知识而不浪费 token？ | ✅ 已完成 | [→](phase2-planning-knowledge/s05-skill-loading.md) | 2026-03-26 |
| 6 | s06 - Context Compact | 上下文窗口满了怎么办？ | ✅ 已完成 | [→](phase2-planning-knowledge/s06-context-compact.md) | 2026-03-26 |
| **第三阶段：持久化** |
| 7 | s07 - Task System | 如何让 Agent 的目标比一次对话更持久？ | ✅ 已完成 | [→](phase3-persistence/s07-task-system.md) | 2026-03-26 |
| 8 | s08 - Background Tasks | Agent 等待慢操作时为什么不能干别的？ | ✅ 已完成 | [→](phase3-persistence/s08-background-tasks.md) | 2026-03-27 |
| **第四阶段：团队** |
| 9 | s09 - Agent Teams | 多个 Agent 如何协同工作？ | ⬜ 未开始 | [→](phase4-teams/s09-agent-teams.md) | — |
| 10 | s10 - Team Protocols | 队友之间怎样进行结构化协商？ | ⬜ 未开始 | [→](phase4-teams/s10-team-protocols.md) | — |
| 11 | s11 - Autonomous Agents | 怎样让 Agent 从"被指派"变成"自己找活干"？ | ⬜ 未开始 | [→](phase4-teams/s11-autonomous-agents.md) | — |
| 12 | s12 - Worktree Isolation | 多个 Agent 同时修改代码怎样不冲突？ | ⬜ 未开始 | [→](phase4-teams/s12-worktree-task-isolation.md) | — |

**状态说明**: ⬜ 未开始 | 🔄 进行中 | ✅ 已完成

---

## 阶段复盘

| 阶段 | 核心主题 | 状态 | 复盘文档 |
|------|----------|------|----------|
| 阶段一 | 感知-推理-行动循环 + 工具扩展 | ⬜ 未开始 | [→](review/phase1-review.md) |
| 阶段二 | 规划 + 隔离 + 知识加载 + 压缩 | ✅ 已完成 | [→](review/phase2-review.md) |
| 阶段三 | 任务持久化 + 异步执行 | ✅ 已完成 | [→](review/phase3-review.md) |
| 阶段四 | 多 Agent 协作 + 自治 + 隔离 | ⬜ 未开始 | [→](review/phase4-review.md) |

---

## 架构演进一览

```
s01: while True + stop_reason          ← 最小可行 Agent
s02: + dispatch map + safe_path        ← 工具扩展（开闭原则）
s03: + TodoManager + nag reminder      ← 对抗注意力稀释
s04: + run_subagent() + 独立messages   ← 上下文隔离
s05: + SkillLoader + YAML frontmatter  ← 两层知识架构
s06: + micro/auto/manual compact       ← 三层压缩策略
s07: + TaskManager + DAG 依赖图        ← 跨会话持久化
s08: + BackgroundManager + 通知队列    ← 异步执行
s09: + TeammateManager + MessageBus    ← 持久身份 + JSONL 邮箱
s10: + FSM + shutdown/plan_approval    ← 结构化协商协议
s11: + idle_poll + 自动认领            ← 自组织 Agent
s12: + WorktreeManager + 事件流        ← 控制平面/执行平面分离
```

---

## 学习规则

1. **按顺序学习**，不跳课（每节课只加一个机制）
2. 每节课结束后立即写笔记并 git commit
3. 完成一个阶段后进行阶段复盘，再进入下一阶段
4. 遇到不懂的概念，先尝试用第一性原理推导，再查文档
