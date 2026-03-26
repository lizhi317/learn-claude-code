# s05 - Skill Loading

> **核心问题**: 怎样让 Agent 拥有领域知识而不浪费 token？
> **参考文档**: `docs/zh/s05-skill-loading.md`
> **参考实现**: `agents/s05_skill_loading.py`
> **状态**: ✅ 已完成
> **学习日期**: 2026-03-26

---

## 一、知识总结

**核心本质**：用两层知识架构实现按需加载——系统提示存摘要（便宜），tool_result 按需注入完整内容（贵），避免把所有领域知识一次性塞进上下文。

**两层架构**：

| 层次 | 位置 | 内容 | token 成本 |
|------|------|------|-----------|
| Layer 1 | system prompt（常驻） | 技能名 + 一句话描述 | ~100/技能 |
| Layer 2 | tool_result（按需） | 完整领域指导 | ~2000/技能 |

**关键代码片段**：

```python
class SkillLoader:
    def __init__(self, skills_dir: Path):
        for f in sorted(skills_dir.rglob("SKILL.md")):
            meta, body = self._parse_frontmatter(f.read_text())
            name = meta.get("name", f.parent.name)  # 目录名兜底
            self.skills[name] = {"meta": meta, "body": body}

# Layer 1：写入系统提示
SYSTEM = f"Skills available:\n{SKILL_LOADER.get_descriptions()}"

# Layer 2：加入 dispatch map，与其他工具接口一致
TOOL_HANDLERS = {
    "load_skill": lambda **kw: SKILL_LOADER.get_content(kw["name"]),
}
```

**架构变更（s04 → s05）**：

| 组件 | s04 | s05 |
|------|-----|-----|
| 新增工具 | `task` | `load_skill` |
| 知识来源 | 无 | `skills/*/SKILL.md` |
| 系统提示 | 静态 | + 技能描述列表 |
| 注入方式 | 无 | 两层（系统提示 + tool_result） |

**可泛化原则**："知道有什么"和"知道具体怎么做"是两种不同粒度的信息，应该分层存储、按需加载。

---

## 二、学习过程反思

**答对的地方**：
- 正确推断两层架构：摘要常驻、详细内容按需
- 理解 `load_skill` 和其他工具复用相同的 dispatch 逻辑

**思维漏洞**：
- 误以为是 Harness 代码驱动技能加载——实际是 LLM 自身根据任务推断后主动调用，Harness 只是提供工具等待调用
- 混淆"不需要更改"和"改动代价极小"——添加工具仍需改 dispatch map 和 TOOLS 列表，只是代价几乎为零

---

## 三、教学引导记录

**产生顿悟的类比**：图书馆索引表——门口的索引告诉你有哪些区（Layer 1），你自己决定去哪个区查资料（LLM 推断），书架上才是完整内容（Layer 2）。

**跨课程线索**：
- s03 的 nag reminder 和 s05 的技能内容都通过 tool_result 注入——这是 Harness 向 LLM 传递信息的通用模式
- s06 Context Compact 将处理"技能加载后上下文依然增长"的后续问题，届时可对比两种策略
