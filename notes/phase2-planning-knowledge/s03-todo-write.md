# s03 - TodoWrite

> **核心问题**: 为什么 Agent 做长任务会"跑偏"？
> **参考文档**: `docs/zh/s03-todo-write.md`
> **参考实现**: `agents/s03_todo_write.py`
> **状态**: ✅ 已完成
> **学习日期**: 2026-03-26

---

## 一、知识总结

**核心本质**：在 Harness 层注入持久化计划状态，对抗 LLM 注意力随上下文增长而稀释的天然缺陷。

**问题根源**：长任务执行过程中，工具调用结果不断填满上下文，系统提示中的原始计划被稀释，LLM 注意力偏离了剩余步骤。任务越长越严重。

**关键设计决策**：

| 设计点 | 选择 | 原因 |
|--------|------|------|
| 同时 in_progress 数量 | 强制 = 1 | LLM 无法自我约束，Harness 硬编码不变量 |
| nag reminder 触发 | >= 3 轮未调用 todo | 平衡提醒频率与上下文噪声 |
| reminder 插入位置 | results 列表最前 | 避免被长工具结果淹没（lost in the middle） |
| render() 返回内容 | 完整清单快照 | 每次更新把所有未完成步骤重新送到 LLM 视野 |

**关键代码片段**：

```python
# 硬约束：同时只能一个 in_progress
if in_progress_count > 1:
    raise ValueError("Only one task can be in_progress")

# nag reminder：连续 3 轮未更新则注入提醒
rounds_since_todo = 0 if used_todo else rounds_since_todo + 1
if rounds_since_todo >= 3:
    results.insert(0, {"type": "text", "text": "<reminder>Update your todos.</reminder>"})
```

**架构变更对比（s02 → s03）**：

| 组件 | s02 | s03 |
|------|-----|-----|
| Tools | 4 | 5（+todo） |
| 规划状态 | 无 | TodoManager（带状态的清单） |
| Loop 附加逻辑 | 无 | rounds_since_todo 计数 + reminder 注入 |

**可泛化原则**：LLM 只能表达意图，Harness 负责保证不变量。凡是"LLM 应该自觉做到"的约束，都应考虑在 Harness 层硬编码。

---

## 二、学习过程反思

**答对的地方**：
- 识别出长任务上下文被工具结果填满是根本问题
- 正确推断 in_progress 同时只能有一个（单 Agent 不需要并发）
- 计数器在工具调用之后递增（第 188 行）

**思维漏洞**：
- 最初把"Agent 遇到测试失败后停下来"归因为"不知道怎么修"（能力问题），而非"忘记还要修"（注意力问题）——这是两种不同的故障模式，需要区分
- 最初猜 render() 只返回当前任务，忽略了"全貌可见"对抗稀释的设计意图——完整清单每次都呈现，才能把被挤出视野的后续步骤拉回来

---

## 三、教学引导记录

**产生顿悟的问题**：
"2 步任务能自己修，5 步任务反而不能——任务难度没变，工具没变，差异在哪？"
→ 这个对比把能力问题和注意力问题拆开，帮助定位真正的变量是上下文长度，而不是 LLM 能力。

**跨课程线索**：
- 学习过程中提出"摘要压缩工具结果"方案 → 这正是 **s06 Context Compact** 的核心机制
- s03（强调未来）vs s06（压缩过去）是对抗注意力稀释的两种互补策略，届时可对比
