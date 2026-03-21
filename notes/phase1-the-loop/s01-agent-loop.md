# s01 - The Agent Loop

> **核心问题**: 一个 AI 模型如何从"只能说话"变成"能做事"？
> **参考文档**: `docs/zh/s01-the-agent-loop.md`
> **参考实现**: `agents/s01_agent_loop.py`
> **状态**: ✅ 已完成
> **学习日期**: 2026-03-21

---

## 一、知识总结

### 本节本质（一句话）

> **Harness 管循环，模型管决策——把工具执行结果喂回模型，让模型自己决定什么时候停。**

### 关键设计决策

| 设计 | 原因 |
|------|------|
| `while True` 而非固定次数 | 任务复杂度不定，模型自己知道什么时候完成 |
| 退出条件用 `!= "tool_use"` | 防御性写法，覆盖 `end_turn`/`max_tokens`/`stop_sequence` 所有退出情况 |
| tool_result 用 `"user"` role | API 规定：工具结果是"外部世界的反馈"，属于 user turn |
| `messages` 累积列表 | 模型无状态，每次调用必须传入完整对话历史 |
| `SYSTEM` 由 Harness 写死 | Agent 的行为准则和能力边界由工程师决定，不是用户，不是模型 |

### 架构变更（s00 → s01）

| 组件 | 之前 | 之后 |
|------|------|------|
| 循环 | 无 | `while True` + stop_reason 退出 |
| 工具 | 无 | bash（单一工具） |
| 消息 | 无 | 累积式 messages 列表 |
| 控制流 | 无 | `stop_reason != "tool_use"` |

### 关键代码

```python
def agent_loop(messages: list):
    while True:
        response = client.messages.create(
            model=MODEL, system=SYSTEM, messages=messages,
            tools=TOOLS, max_tokens=8000,
        )
        messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason != "tool_use":   # 防御性退出
            return

        results = []
        for block in response.content:
            if block.type == "tool_use":
                output = run_bash(block.input["command"])
                results.append({"type": "tool_result", "tool_use_id": block.id,
                                "content": output})
        messages.append({"role": "user", "content": results})  # user role
```

### stop_reason 取值

| 值 | 含义 | 循环行为 |
|----|------|----------|
| `"tool_use"` | 模型要调用工具 | 继续 |
| `"end_turn"` | 模型正常说完 | 退出 |
| `"max_tokens"` | token 用完被截断 | 退出 |
| `"stop_sequence"` | 触发停止词 | 退出 |

### 可泛化原则

- **防御性退出条件**（排除法）比正向枚举更健壮，适用于任何状态机设计
- **职责分离**：框架管流程，智能体管决策——在微服务、工作流引擎中同样适用

---

## 二、学习过程反思

### 答对的地方

- 题 1 结论正确：`!= "tool_use"` 比 `== "end_turn"` 更安全
- 题 2 正确识别了上下文爆炸问题，猜到 s06 解决这个问题
- 题 3 正确理解了去掉 `"Act, don't explain"` 的影响，以及 Harness 工程师控制 Agent 行为准则的设计含义

### 思维漏洞

**题 1**：误以为 token 用完会"拿不到返回值"。
- 错误理解：token 耗尽 → API 无返回值
- 正确理解：API 依然返回响应，`stop_reason = "max_tokens"`，内容只是截断的。若用 `== "end_turn"` 作退出条件，此时不满足条件，会死循环。

**题 2**：判断方向反了，认为"用户指令优先级更高不会被稀释"。
- 错误理解：用户指令重要，注意力不会稀释它
- 正确理解：Transformer 注意力偏向最近内容，**最先被稀释的恰恰是最早的用户指令**，工具结果因为在末尾反而更受关注

### 模糊地带

- `stop_reason` 的完整取值范围需要记住（见上表）

---

## 三、教学引导记录

### 产生顿悟的类比

"不离开办公桌的专家顾问"——把抽象的 `while True` 具象化为日常协作场景：顾问写便条 → 你去执行 → 带结果回来 → 循环，直到顾问不再要求。瞬间理解了"为什么需要循环"。

### 跨课程线索（后续关注）

| 问题 | 解决课程 |
|------|----------|
| 上下文越长，早期用户指令越被稀释 | s03 TodoWrite（注入任务清单对抗注意力衰减） |
| messages 无限增长，上下文爆炸 | s06 Context Compact（三层压缩策略） |
| Harness 控制 Agent 行为准则 | s05 Skill Loading（动态注入领域知识） |
