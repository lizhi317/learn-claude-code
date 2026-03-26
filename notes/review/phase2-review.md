# 阶段二复盘：规划与知识（s03-s06）

> **完成日期**: 2026-03-26
> **状态**: ✅ 已完成

---

## 费曼检验：一句话解释四个机制

**s03 - TodoWrite**：给 Agent 一份执行清单，知道所有任务的执行情况，并在需要更新的时候 Harness 提醒 Agent 更新。

**s04 - Subagent**：父 Agent 拆解任务，将单一的小任务分配给子 Agent 执行，只关心执行的返回结果，子 Agent 用后即弃。

**s05 - Skill Loading**：将 Agent 需要的外部知识通过 skill 形式封装，按需加载；第一层只提取名称和描述，需要时再将完整技能加载到上下文。

**s06 - Context Compact**：前几个策略是在避免问题，上下文压缩的本质是解决问题——保留最近 N 次工具调用（micro），或本地存档后 LLM 摘要替换（auto/manual）。

---

## 深度复盘：三个反复出现的思维漏洞

### 模式一：混淆"谁是主动方"

**出现位置**：s03（以为 Harness 自己更新 TodoList）、s05（以为 Harness 驱动技能加载）

**正确模型**：

```
Harness 的角色         LLM 的角色
──────────────         ──────────
提供工具               决定调用哪个工具
注入提醒               决定如何响应提醒
设置阈值               做出判断和推理
```

Harness 永远是被动的——只在 LLM 发出 `tool_use` 后才响应。注入操作（reminder、skill 描述）只是把信息送到 LLM 眼前，决定怎么用的是 LLM。

**工程类比**：Spring IoC 容器提供 Bean、管理生命周期，但业务逻辑由代码自己决定怎么跑。容器是基础设施，不是决策者。

---

### 模式二：消息协议理解不系统

**出现位置**：s04（子 Agent 摘要类型）、s06（压缩后消息条数）

**完整协议图**：

```
assistant message:
  content: [
    { type: "text",     text: "..." },                              ← LLM 说明（可选）
    { type: "tool_use", id: "x1", name: "read_file", input: {...} } ← LLM 请求
  ]

user message（Harness 构造）:
  content: [
    { type: "tool_result", tool_use_id: "x1", content: "..." }      ← Harness 响应
  ]
```

**两个硬性约束**：
1. `tool_use` 和 `tool_result` 必须通过 `tool_use_id` 一一对应
2. messages 必须 user/assistant 严格交替——API 硬性要求

子 Agent 摘要是 `tool_result`（Harness 响应工具调用）；压缩后必须留两条消息（维持交替结构）——两个问题同一个根因。

---

### 模式三：二元思维 vs 分层思维

**出现位置**：s06（想清掉所有工具结果）、s03（以为 render() 只返回当前任务）

极端操作（全清/全留）往往都错，正确答案在分层中间：

| 信息 | 时间越远 | 策略 |
|------|---------|------|
| 工具结果详情 | 价值越低 | 替换为占位符 |
| 事件发生的事实 | 始终有用 | 保留占位符 |
| TodoList 全貌 | 始终有用 | 每次完整渲染 |

**工程类比**：缓存策略从不是"全缓存"或"不缓存"，而是 LRU、TTL 等分层策略——和 micro_compact 逻辑如出一辙。

---

## 四机制协同场景

**任务**："重构整个项目，加类型注解、写单测、提交 git"

```
Agent 启动
│
├─ [s03 TodoList] 列出 5 个步骤，标记 step1 in_progress
│
├─ [s05 Skill] 调用 load_skill("git") 加载 git 工作流指导
│
├─ [s04 Subagent] 派子 Agent 扫描所有 .py 文件，返回摘要
│   └── 子 Agent 读了 20 个文件，messages 用完即扔
│       父 Agent 只收到："共 8 个文件需要重构"
│
├─ [s03 nag] 3 轮后未更新 Todo → 注入 <reminder>
│   └── Agent 更新：step1 completed，step2 in_progress
│
├─ [s06 micro_compact] 旧的工具结果静默替换为占位符
│
├─ ... 继续执行 step3, step4 ...
│
├─ [s06 auto_compact] token 超 50000 → 保存 transcript，LLM 摘要
│   └── messages 压缩为 2 条，继续执行
│
└─ [s03 TodoList] 所有步骤 completed，Agent 结束
```

四个机制各司其职：
- **s03** 保证不跑偏
- **s04** 保证子任务不污染主上下文
- **s05** 保证知识按需加载不浪费
- **s06** 保证上下文不爆满能持续工作

---

## 架构演进图

```
s02: while True + dispatch map
      ↓ +TodoManager + nag reminder
s03: + rounds_since_todo 计数器
      ↓ +run_subagent() + 独立 messages[]
s04: + task 工具（仅父端）
      ↓ +SkillLoader + YAML frontmatter
s05: + load_skill 工具 + 两层知识架构
      ↓ +micro/auto/manual compact
s06: + compact 工具 + .transcripts/ 存档
```

---

## 第二阶段可泛化原则

| 原则 | 在 s03-s06 的体现 |
|------|-----------------|
| Harness 保证不变量，LLM 做决策 | 硬约束 in_progress=1、禁止递归派生 |
| 分层管理不同粒度的信息 | skill 两层、micro/auto/manual 三层 |
| 基础设施被动响应，不主动干预决策 | Harness 只在 tool_use 后响应 |
| 信息移出活跃区 ≠ 丢失 | transcript 存档、占位符保留事实 |
| 极端操作往往错，正确答案在分层中间 | 选择性清理而非全清 |
