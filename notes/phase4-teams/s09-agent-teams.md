# s09 - Agent Teams

> **核心问题**: 多个 Agent 如何协同工作？
> **参考文档**: `docs/zh/s09-agent-teams.md`
> **参考实现**: `agents/s09_agent_teams.py`
> **状态**: ✅ 已完成

---

## 一、知识总结

### 本节核心概念

**一句话本质**：给每个 Agent 持久身份（TeammateManager + config.json）和专属邮箱（MessageBus + JSONL），让多个 Agent 并行工作、异步通信，队友不再用完即扔。

### 关键设计决策表

| 问题 | 决策 | 原因 |
|------|------|------|
| 队友信息存哪里？ | `config.json` 名册 | 中央注册表，任意 Agent 可查询谁在团队、谁空闲 |
| Agent 间如何通信？ | JSONL 邮箱（每人一个文件） | 追加写天然支持，无需解析整个文件，适合并发写入 |
| 为什么用 JSONL 不用大 JSON？ | 追加写 vs 解析重写 | 大 JSON 文件追加需要读取→修改→重写整个文件 |
| 消息何时被感知？ | 每次 LLM 调用前 read_inbox | 与 s08 通知队列同一模式：调用前注入，LLM 决策时有完整上下文 |
| 为什么读完清空？ | 消费即确认，不重复投递 | 不清空会导致每轮重复处理同一条消息 |
| 为什么优先用 idle 队友而非新 spawn？ | 复用线程和历史上下文 | 省资源、保留工作记忆、身份连续不重复 |

### 核心架构

```
.team/
  config.json          ← 团队名册（name, role, status）
  inbox/
    alice.jsonl        ← alice 的收件箱（append-only, drain-on-read）
    bob.jsonl
    lead.jsonl

队友生命周期：
  spawn → WORKING → IDLE → WORKING → ... → SHUTDOWN
```

### 关键代码片段

**spawn：创建持久队友**
```python
def spawn(self, name: str, role: str, prompt: str) -> str:
    member = {"name": name, "role": role, "status": "working"}
    self.config["members"].append(member)   # 写入名册
    self._save_config()
    thread = threading.Thread(
        target=self._teammate_loop,
        args=(name, role, prompt), daemon=True)
    thread.start()
    return f"Spawned teammate '{name}'"
```

**MessageBus：追加写 + 读完清空**
```python
def send(self, sender, to, content, ...):
    with open(self.dir / f"{to}.jsonl", "a") as f:
        f.write(json.dumps(msg) + "\n")    # 追加一行

def read_inbox(self, name):
    msgs = [json.loads(l) for l in path.read_text().splitlines()]
    path.write_text("")   # drain（读完清空）
    return json.dumps(msgs)
```

**队友循环：每轮先检查收件箱**
```python
def _teammate_loop(self, name, role, prompt):
    messages = [{"role": "user", "content": prompt}]
    for _ in range(50):
        inbox = BUS.read_inbox(name)
        if inbox != "[]":
            messages.append({"role": "user", "content": f"<inbox>{inbox}</inbox>"})
            messages.append({"role": "assistant", "content": "Noted inbox messages."})
        response = client.messages.create(...)
        if response.stop_reason != "tool_use":
            break
    self._find_member(name)["status"] = "idle"  # 任务完成→空闲
```

### 三种 Agent 机制对比

| | s04 子 Agent | s08 后台任务 | s09 队友 |
|--|-------------|-------------|---------|
| 执行内容 | LLM 推理 + 工具 | shell 命令 | LLM 推理 + 工具 |
| 生命周期 | 用完即扔 | 命令跑完销毁 | idle→working 持久循环 |
| 身份 | 无 | 无 | 持久（config.json） |
| 父级关系 | 父 Agent 等待 | 并行，通知队列 | 并行，邮箱通信 |
| 通信方式 | 直接返回结果 | 推入通知队列 | JSONL 邮箱 |

### 架构变更对比

| 组件 | s08 之前 | s09 之后 |
|------|----------|----------|
| Agent 数量 | 单一 | 领导 + N 个队友 |
| 持久化 | 无 | config.json + JSONL 收件箱 |
| 线程 | 后台命令 | 每线程完整 agent loop |
| 生命周期 | 一次性 | idle → working → idle |
| 通信 | 无 | send / broadcast / read_inbox |

### 可泛化原则

- **邮箱模式**：Actor 模型（Erlang/Akka）、微服务消息队列（Kafka）、电子邮件系统——本质都是"每个执行者有专属队列，异步通信，读完清空"
- **外部变化感知**：外部世界的变化（消息、后台结果）通过 LLM 调用前注入上下文来感知——s08 和 s09 的共同模式
- **持久身份 vs 用完即扔**：长期协作用持久身份（节省资源、保留上下文）；一次性隔离任务用子 Agent（防污染）

---

## 二、学习过程反思

### 答对的地方

- 正确推导出"中央注册表"（config.json）和"邮箱机制"（JSONL）两个核心机制
- 正确理解 JSONL 追加写的优势
- 正确理解读完清空的语义（消费即确认）
- 正确识别 idle 状态的意义（持久身份，等待下一个任务）

### 犯的错误或思维漏洞

- **为什么不 spawn 新队友**：最初只想到成本和管控问题，忽略了最直接的原因——idle 队友保留了历史上下文和连续身份，新 spawn 的队友什么都不知道

### 思考不够清晰的地方

- 收件箱消息注入时机（每次 LLM 调用前）和 s08 通知队列是同一个模式，当时需要提示才想起来，需要加深这个跨课程模式的记忆

---

## 三、教学引导记录

### 产生顿悟的类比/问题

- "入职第一天怎么认识同事"→"人事部（config.json）+ 邮箱（JSONL）"——让抽象的多 Agent 协调机制立刻有了直觉

- s04/s08/s09 三种机制的对比表——第一次把三者放在一起比较，清晰看出"用完即扔 vs 持久循环"的本质差异

### 需要后续关注的跨课程线索

- s10 Team Protocols：多 Agent 之间如何进行结构化协商（FSM + shutdown/plan_approval）
- 邮箱模式下如何防止消息丢失（s09 的 drain 是破坏性读取，crash 后消息丢失怎么办）

---

*学习日期：2026-03-29*
*提交 commit：待提交*
