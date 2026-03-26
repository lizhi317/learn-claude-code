# s06 - Context Compact

> **核心问题**: 上下文窗口满了怎么办？
> **参考文档**: `docs/zh/s06-context-compact.md`
> **参考实现**: `agents/s06_context_compact.py`
> **状态**: ✅ 已完成
> **学习日期**: 2026-03-26

---

## 一、知识总结

**核心本质**：上下文窗口是硬上限，三层递进压缩策略让 Agent 可以无限期工作——轻量占位、LLM 摘要、主动触发三种手段各司其职。

**三层压缩**：

| 层次 | 触发方式 | 手段 | LLM 参与 |
|------|---------|------|---------|
| micro_compact | 每轮静默 | 旧 tool_result → 占位符 | 否 |
| auto_compact | token > 50000 | LLM 摘要 + 存磁盘 | 是 |
| compact 工具 | LLM 主动调用 | 同 auto_compact | 是 |

**关键代码片段**：

```python
# Layer 1：每轮静默替换旧工具结果
def micro_compact(messages):
    # 保留最近 KEEP_RECENT 条，更早的替换为占位符
    for _, _, part in tool_results[:-KEEP_RECENT]:
        if len(part.get("content", "")) > 100:
            part["content"] = f"[Previous: used {tool_name}]"

# Layer 2：token 超阈值自动压缩
def auto_compact(messages):
    # 1. 先存档到磁盘
    with open(transcript_path, "w") as f:
        for msg in messages: f.write(json.dumps(msg) + "\n")
    # 2. LLM 做摘要，返回两条消息（维持 user/assistant 交替格式）
    return [
        {"role": "user",      "content": "[Compressed]\n\n{摘要}"},
        {"role": "assistant", "content": "Understood. Continuing."},
    ]

# Loop 整合三层
def agent_loop(messages):
    while True:
        micro_compact(messages)                      # Layer 1，每轮
        if estimate_tokens(messages) > THRESHOLD:
            messages[:] = auto_compact(messages)     # Layer 2，超阈值
        response = client.messages.create(...)
        if manual_compact:
            messages[:] = auto_compact(messages)     # Layer 3，主动触发
```

**关键设计细节**：
- 占位符保留而不删除：保留事件历史事实 + 避免 `tool_use` 无对应 `tool_result` 的 API 报错
- 压缩前先存 transcript：信息不丢失，只是移出活跃上下文，可审计可恢复
- 压缩后保留两条消息：满足 API user/assistant 交替格式要求，循环无缝继续

**架构变更（s05 → s06）**：

| 组件 | s05 | s06 |
|------|-----|-----|
| 上下文管理 | 无 | 三层压缩 |
| 新增工具 | `load_skill` | `compact` |
| Loop 附加逻辑 | 无 | 每轮 micro_compact + token 检测 |
| 持久化 | 无 | `.transcripts/` 存档 |

**可泛化原则**：信息不需要永远活跃，移出活跃上下文 ≠ 丢失。分层管理"热数据"和"冷数据"是应对容量限制的通用模式。

---

## 二、学习过程反思

**答对的地方**：
- 正确识别压缩的核心手段是清理旧工具调用结果
- 理解存 transcript 的两个原因：防随机故障 + 保留可查原始记录
- 理解第三层是给 LLM 主动整理的能力，Harness 兜底 + LLM 自主两条路

**思维漏洞**：
- 最初想"清掉所有工具调用结果"——忽略了最近的结果仍然重要，应该按时间远近而非"中间/最终"来判断
- 误以为 auto_compact 压缩后只需保留摘要一条——忽略了 API 消息必须 user/assistant 交替的结构要求

---

## 三、教学引导记录

**关键问题**："为什么不直接删除占位符，而是留着？"——让我意识到 micro_compact 同时解决了两个问题：保留历史事实 + 维持 API 消息格式合法性。一个设计决策同时满足两个约束。

**跨课程线索**：
- s03 对抗"忘记未来步骤"，s06 对抗"被过去内容填满"——两者互补，共同守护 Agent 的注意力质量
- s05（按需加载）+ s06（主动压缩）是第二阶段的完整解法：一个控制知识注入，一个控制历史积累
