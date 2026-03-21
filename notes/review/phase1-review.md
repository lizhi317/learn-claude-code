# 阶段一复盘：循环（s01-s02）

> **完成日期**: 2026-03-21
> **状态**: ✅ 已完成

---

## 费曼三问（不看笔记作答）

**1. Harness 是什么？它和模型的分工？**

> Harness 管循环——维持 loop、提供工具、执行工具结果回传；模型管决策——推理、选工具、决定什么时候停止。
>
> 一句话：**Harness 管循环，模型管决策。**

**2. s01 → s02 新增了什么？**

> 三件事，循环骨架一行未动：
> - dispatch map（字典路由）：加工具不动循环
> - safe_path（路径沙箱）：防止路径逃逸工作目录
> - 错误返回字符串：循环遇错不崩，模型自己处理

**3. s01 埋下的两个定时炸弹？**

> - 上下文爆炸 + 早期指令被稀释 → s03 TodoWrite / s06 Context Compact
> - 没有领域知识加载机制 → s05 Skill Loading

---

## 架构演进图

```
s01:
while True:
    response = LLM(messages)
    if stop: return
    output = run_bash(command)      ← 硬编码，只有一条路
    append(tool_result)

s02:
while True:
    response = LLM(messages)
    if stop: return
    handler = TOOL_HANDLERS[name]   ← 字典查找，N 条路
    output = handler(**input)
    append(tool_result)
```

> **循环骨架一模一样，只有工具分发那一行变了。**

---

## 阶段一可泛化原则

| 原则 | 在 s01-s02 的体现 | 其他领域应用 |
|------|-------------------|-------------|
| 防御性退出（排除法） | `stop_reason != "tool_use"` | 任何状态机的退出条件 |
| 开闭原则 | dispatch map，加工具不动循环 | Spring 路由、插件系统 |
| 职责分离 | Harness 管流程，模型管决策 | 微服务、工作流引擎 |
| 错误转数据 | 未知工具返回字符串保持循环存活 | 任何需要保持流程存活的系统 |
