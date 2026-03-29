# s10 - Team Protocols

> **核心问题**: 队友之间怎样进行结构化协商？
> **参考文档**: `docs/zh/s10-team-protocols.md`
> **参考实现**: `agents/s10_team_protocols.py`
> **状态**: ✅ 已完成

---

## 一、知识总结

### 本节核心概念

**一句话本质**：多 Agent 自由通信缺少结构化协商协议，通过引入 FSM（pending → approved/rejected）和 request_id 关联机制，让关机和高风险操作都有明确的握手流程。

### s07 vs s10 的问题区分

| | s07 | s10 |
|--|-----|-----|
| **问题** | 任务进度无法跨会话保存 | Agent 间协商缺少结构化规则 |
| **场景** | "做完第 237 个，下次从哪继续？" | "关掉 Alice，文件写到一半怎么办？" |
| **解决手段** | 任务状态持久化到磁盘 DAG | 请求-响应 FSM，结构化握手 |

### 关键设计决策表

| 问题 | 决策 | 原因 |
|------|------|------|
| 如何关联请求和响应？ | `request_id` 唯一标识 | 多个请求并发时，响应必须能找到对应的请求 |
| 为什么不直接杀线程？ | 关机握手协议 | 直接杀：文件写一半损坏 + config.json 状态永远是 working |
| 为什么队友发起计划审批？ | 执行细节由队友制定 | 领导审的是队友的方案，不是自己写方案发给队友 |
| 两种协议为何用同一个 FSM？ | 结构相同，内容不同 | 请求-响应模式可抽象复用，pending→approved/rejected 适用所有协商场景 |

### 核心 FSM

```
[pending] --approve--> [approved]
[pending] --reject---> [rejected]
```

**关机协议流程：**
```
Lead                    Teammate
  |--shutdown_req(id)-->|
  |                     | （收尾，写完文件）
  |<--shutdown_resp(id)-|
  |   {approve: true}   |
  |                     | → status: shutdown
```

**计划审批流程：**
```
Teammate               Lead
  |--plan_req(id)------>|
  |                     | （审查计划）
  |<--plan_resp(id)-----|
  |   {approve: true}   |
  |                     | → 开始执行
```

### 关键代码片段

**关机请求：生成 req_id，存入追踪表，发送消息**
```python
def handle_shutdown_request(teammate: str) -> str:
    req_id = str(uuid.uuid4())[:8]
    shutdown_requests[req_id] = {"target": teammate, "status": "pending"}
    BUS.send("lead", teammate, "Please shut down gracefully.",
             "shutdown_request", {"request_id": req_id})
    return f"Shutdown request {req_id} sent (status: pending)"
```

**收到响应：引用同一个 req_id，更新状态**
```python
shutdown_requests[req_id]["status"] = "approved" if approve else "rejected"
```

### 架构变更对比

| 组件 | s09 之前 | s10 之后 |
|------|----------|----------|
| 关机 | 仅自然退出（直接杀线程） | 请求-响应握手 |
| 计划门控 | 无 | 提交/审查与审批 |
| 关联机制 | 无 | 每个请求一个 request_id |
| FSM | 无 | pending → approved/rejected |
| 工具数量 | 9 | 12 (+shutdown_req/resp +plan) |

### 可泛化原则

- **请求-响应模式**：HTTP、RPC、数据库查询、s08 后台任务的 task_id——所有异步操作都需要关联键把请求和响应绑定
- **FSM 抽象复用**：结构相同的业务场景（关机/计划审批/代码审查/资源申请）可以共用同一个状态机，只是消息内容不同
- **高风险操作门控**：执行前审批，避免返工——CI/CD 的 manual approval、数据库 DDL 的 review 流程都是同一个模式
- **开闭原则**：新增 3 个工具，核心循环零改动（s02 建立，贯穿全课）

---

## 二、学习过程反思

### 答对的地方

- 正确联想到计算机网络中的确认机制
- 正确理解 req_id 用于追踪多个并发请求的响应
- 正确推理出直接杀线程导致"文件写一半"的后果
- 正确理解计划审批的门控语义（高风险操作前审批）

### 犯的错误或思维漏洞

- **s10 解决的问题**：最初描述为"不知道任务状态执行到哪"——这是 s07 的问题。s10 解决的是"多 Agent 协商缺少结构化协议"，两者容易混淆，需要从场景出发区分。
- **FSM 状态**：最初说"开启、执行、关闭"，正确是 `pending → approved | rejected`——描述的是协商状态，不是任务执行状态。
- **直接杀线程的后果**：只想到文件保存问题，需要引导才想到 config.json 状态过期——"调用方感知不到被调用方已死"是分布式系统中的经典问题，需要加深印象。

### 思考不够清晰的地方

- 开闭原则每节课都在用，但名字还是记不住，需要反复强化

---

## 三、教学引导记录

### 产生顿悟的类比/问题

- "交通信号灯：红→绿→黄→红，每个灯是一个状态，切换有明确规则"——让 FSM 从抽象概念变成了直觉

- s07 vs s10 的对比表——"目标在哪 vs 协商怎么做"，一句话把两节课的问题边界划清楚了

### 需要后续关注的跨课程线索

- s11 Autonomous Agents：Agent 从"被分配任务"变成"自己找活干"，FSM 协议在自治场景下如何演变
- s12 Worktree Isolation：多 Agent 同时修改代码时，关机协议和计划审批如何配合避免代码冲突

---

*学习日期：2026-03-29*
*提交 commit：待提交*
