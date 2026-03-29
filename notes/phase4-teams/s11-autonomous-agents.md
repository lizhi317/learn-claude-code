# s11 - Autonomous Agents

> **核心问题**: 怎样让 Agent 从"被指派"变成"自己找活干"？
> **参考文档**: `docs/zh/s11-autonomous-agents.md`
> **参考实现**: `agents/s11_autonomous_agents.py`
> **状态**: ✅ 已完成

---

## 一、知识总结

### 本节核心概念

**一句话本质**：队友 Agent 变成 `idle` 后主动轮询任务图，自动认领无前置依赖的未分配任务，不再依赖领导指派——自组织团队。

### 关键设计决策表

| 问题 | 决策 | 原因 |
|------|------|------|
| 如何自动认领任务？ | idle_poll：每 5 秒扫描任务图 | 拉模式，不依赖外部推送，队友自治 |
| 为什么用拉模式不用推模式？ | 没有天然发送方 | 推模式需要领导推送，违背自组织初衷 |
| 为什么设置 60 秒超时关机？ | 释放线程/内存资源 | 任务做完空转浪费资源，及时回收 |
| 为什么需要身份重注入？ | 压缩后 messages 丢失身份信息 | s11 队友长期运行，auto_compact 必然触发 |
| 认领条件是什么？ | status=pending + owner 为空 + blockedBy 为空 | 可执行 + 未认领 + 无前置依赖 |
| 并发认领有锁吗？ | 没有 | 教学实现，最坏结果是重复执行，不是数据损坏 |

### 队友生命周期

```
+-------+
| spawn |
+---+---+
    |
    v
+-------+   tool_use     +-------+
| WORK  | <------------- |  LLM  |
+---+---+                +-------+
    |
    | stop_reason != tool_use
    v
+--------+
|  IDLE  |  poll every 5s for up to 60s
+---+----+
    |
    +---> check inbox → message? ---------> WORK
    |
    +---> scan .tasks/ → unclaimed? -----> claim → WORK
    |
    +---> 60s timeout ------------------> SHUTDOWN
```

### 推模式 vs 拉模式（重要思考路径）

| | 推模式（s08） | 拉模式（s11） |
|--|-------------|-------------|
| 触发方式 | 事件发生时主动推入队列 | 定时主动扫描 |
| 发送方 | 后台线程（天然知道自己完成了） | 无发送方 |
| 适用场景 | 有明确的事件源 | 需要自治，无外部推送方 |
| 类比 | 微信消息推送 | 邮件客户端定时收信 |

**选拉模式的根本原因**：s11 的设计目标是去掉领导分配，如果用推模式，就需要某个主体在有新任务时主动通知队友——那个主体只能是领导，违背自组织初衷。

### 身份重注入机制

**为什么 s11 之前不需要：**
- s01-s08：单一 Agent，system prompt 始终有效
- s09/s10：队友生命周期短，很少触发 auto_compact

**为什么 s11 需要：**
s11 队友持续执行多个任务，经历多个 idle→working 循环，messages 不断积累，必然触发 auto_compact（s06），压缩后 messages 只剩 2 条摘要，身份信息丢失。

**检测方式：**
```python
if len(messages) <= 3:   # 说明发生了压缩
    messages.insert(0, {"role": "user",
        "content": f"<identity>You are '{name}', role: {role}, "
                   f"team: {team_name}.</identity>"})
    messages.insert(1, {"role": "assistant",
        "content": f"I am {name}. Continuing."})
```

### 关键代码片段

**idle_poll：空闲时拉取任务**
```python
def _idle_poll(self, name, messages):
    for _ in range(12):              # 60s / 5s = 12 次
        time.sleep(5)
        inbox = BUS.read_inbox(name)
        if inbox:
            messages.append({"role": "user", "content": f"<inbox>{inbox}</inbox>"})
            return True
        unclaimed = scan_unclaimed_tasks()
        if unclaimed:
            claim_task(unclaimed[0]["id"], name)
            messages.append({"role": "user",
                "content": f"<auto-claimed>Task #{unclaimed[0]['id']}: ..."})
            return True
    return False   # 超时 → 关机
```

**scan_unclaimed_tasks：三个条件**
```python
if (task.get("status") == "pending"
        and not task.get("owner")
        and not task.get("blockedBy")):
    unclaimed.append(task)
```

### 架构变更对比

| 组件 | s10 之前 | s11 之后 |
|------|----------|----------|
| 自治性 | 领导指派 | 自组织 |
| 空闲阶段 | 无 | 轮询收件箱 + 任务看板 |
| 任务认领 | 仅手动 | 自动认领未分配任务 |
| 身份 | 系统提示 | + 压缩后重注入 |
| 超时 | 无 | 60 秒空闲 → 自动关机 |

### 可泛化原则

- **推/拉模式的选择**：有天然事件源用推（消息队列、webhook）；需要自治无发送方用拉（定时任务、爬虫、轮询）
- **资源及时回收**：空闲超时关机 = 连接池回收空闲连接、K8s HPA 缩容——长期空转是资源浪费
- **身份锚点**：分布式系统中 Agent/进程重启或上下文丢失后，必须有机制重建身份和状态（会话恢复、JWT 重签）

---

## 二、学习过程反思

### 答对的地方

- 正确推导出"队友变 idle 后扫描任务图，找无前置依赖的任务认领"的完整机制
- 正确理解 60 秒超时关机是资源管理的考量
- 在引导下正确理解推/拉模式的区别，并理解 s11 选拉模式的根本原因
- 正确理解身份重注入的触发条件（auto_compact 后 messages 变短）

### 犯的错误或思维漏洞

- **身份重注入的原因**：最初理解为"需要知道身份才能认领相关任务"，偏离了根本原因。正确理解是：s11 队友长期运行必然触发压缩，压缩后 messages 丢失身份信息，LLM 行为会漂移，需要重注入身份锚点。

- **推模式的本质**：最初只记得"s08 用通知队列"，需要看代码才想起"后台线程完成后主动推入"是事件驱动，而不是轮询。推/拉的区别需要在脑中建立更清晰的模型。

### 思考不够清晰的地方

- 并发认领的竞态条件：看到多线程同时认领，自然反应是"加锁"，但 s11 没有加锁——教学实现接受最坏结果是"重复执行"而非"数据损坏"，这种设计取舍需要加深理解

---

## 三、教学引导记录

### 产生顿悟的类比/问题

- "如果用推模式，谁来推？"——这个问题一下子让推/拉模式的选择有了逻辑必然性，而不是随意的技术选型

- 推/拉模式对比表（微信推送 vs 邮件定时收信）——把抽象的模式映射到日常经验

### 需要后续关注的跨课程线索

- s12 Worktree Isolation：多 Agent 自动认领任务后同时修改代码，如何隔离避免冲突
- 生产级并发认领：Redis SET NX / 数据库 SELECT FOR UPDATE——s11 的教学简化和生产实现的差距

---

*学习日期：2026-03-29*
*提交 commit：待提交*
