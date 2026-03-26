# s04 - Subagent

> **核心问题**: 为什么需要"用完即扔"的子智能体？
> **参考文档**: `docs/zh/s04-subagent.md`
> **参考实现**: `agents/s04_subagent.py`
> **状态**: ✅ 已完成
> **学习日期**: 2026-03-26

---

## 一、知识总结

**核心本质**：给子任务分配独立的 messages 数组，执行完毕后只返回摘要，上下文用完即扔，避免中间结果污染父 Agent 的上下文。

**关键设计决策**：

| 设计点 | 选择 | 原因 |
|--------|------|------|
| sub_messages 作用域 | 函数局部变量 | 函数返回后自动销毁，用语言机制保证隔离 |
| 子 Agent 无 `task` 工具 | 硬切断 | 防止递归派生，Harness 层保证不变量 |
| 安全循环上限 `range(30)` | 保险丝 | 子 Agent 失控时有兜底，防止无限 API 调用 |
| 返回值类型 | `tool_result` | 子 Agent 对父 Agent 是普通工具，接口统一 |

**关键代码片段**：

```python
def run_subagent(prompt: str) -> str:
    sub_messages = [{"role": "user", "content": prompt}]  # 独立上下文，从空列表开始
    for _ in range(30):  # 保险丝：最多 30 轮
        response = client.messages.create(...)
        sub_messages.append(...)
        if response.stop_reason != "tool_use":
            break
        # ... 处理工具调用 ...
    return "".join(b.text for b in response.content if hasattr(b, "text")) or "(no summary)"
    # sub_messages 函数返回后自动销毁
```

**架构变更（s03 → s04）**：

| 组件 | s03 | s04 |
|------|-----|-----|
| 上下文 | 单一共享 | 父/子隔离 |
| 新增工具 | — | `task`（仅父端） |
| 新增函数 | — | `run_subagent()` |
| 子 Agent 返回值 | — | `tool_result`（摘要字符串） |

**可泛化原则**：Prompt 是建议，工具列表是能力边界，Harness 是唯一可信的约束层。

---

## 二、学习过程反思

**答对的地方**：
- 识别出子 Agent 用完即扔、不需要持久性
- 理解禁止递归派生的原因：防止无限派生调用链失控
- 理解 `range(30)` 是防止单个子 Agent 内部死循环的保险丝

**思维漏洞**：
- 混淆了 s03 的 `todo` 工具和 s04 的 `task` 工具——`task` 是派生子 Agent 的入口，不是任务清单
- 误以为子 Agent 摘要以 `text` 类型返回父 Agent，实际是 `tool_result`——子 Agent 对父 Agent 来说只是一个普通工具，与读文件、跑命令的返回形式完全一样

---

## 三、教学引导记录

**产生顿悟的类比**：项目经理 vs 实习生——实习生查完资料只汇报结论，草稿纸扔掉。这把"上下文隔离"的设计意图说得很清楚：父 Agent 只需要结果，不需要过程。

**跨课程线索**：
- s04 的子 Agent 上下文是"执行完即丢"的临时隔离（内存层面）
- s12 Worktree Isolation 将在文件系统层面做类似的隔离
- 届时可对比：内存隔离 vs 文件系统隔离，两种隔离粒度和用途的差异
