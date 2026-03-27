# s08 - Background Tasks

> **核心问题**: Agent 等待慢操作时为什么不能干别的？
> **参考文档**: `docs/zh/s08-background-tasks.md`
> **参考实现**: `agents/s08_background_tasks.py`
> **状态**: ✅ 已完成

---

## 一、知识总结

### 本节核心概念

**一句话本质**：把慢 I/O 操作（shell 命令）扔到后台线程，完成后推入通知队列，主 Agent 循环每次 LLM 调用前排空队列——LLM 不阻塞，I/O 并行化。

### 关键设计决策表

| 问题 | 决策 | 原因 |
|------|------|------|
| 如何不阻塞主循环？ | 后台线程立即返回 task_id | 主循环继续执行，I/O 在子线程等待 |
| 后台任务完成后如何通知？ | 推入 `_notification_queue` | 不轮询，事件驱动，完成时主动推 |
| 何时消费通知队列？ | 每次 LLM 调用前排空 | LLM 决策时需要完整上下文，调用后注入无效 |
| 为什么用 daemon 线程？ | 生命周期绑定主进程 | 主进程退出时不留僵尸线程 |
| 为什么加锁？ | 多线程并发写队列需互斥 | 不加锁会数据竞争，结果损坏或丢失 |
| 为什么要 timeout=300？ | 防止命令卡死变僵尸线程 | 无超时则队列永远无结果，资源泄漏 |

### 核心架构

```
Main thread                Background thread
+-----------------+        +-----------------+
| agent loop      |        | subprocess runs |
| drain queue <---+--------| enqueue(result) |
| LLM call        |        +-----------------+
+-----------------+

Timeline:
Agent --[spawn A]--[spawn B]--[other work]----
             |          |
             v          v
          [A runs]   [B runs]      (parallel)
             |          |
             +-- results injected before next LLM call
```

### 关键代码片段

**立即返回，后台跑：**
```python
def run(self, command: str) -> str:
    task_id = str(uuid.uuid4())[:8]
    thread = threading.Thread(
        target=self._execute, args=(task_id, command), daemon=True)
    thread.start()
    return f"Background task {task_id} started"  # 立即返回
```

**完成后推入队列：**
```python
def _execute(self, task_id, command):
    r = subprocess.run(command, shell=True, timeout=300, ...)
    with self._lock:
        self._notification_queue.append({"task_id": task_id, "result": output})
```

**LLM 调用前排空：**
```python
def agent_loop(messages):
    while True:
        notifs = BG.drain_notifications()   # ← 每轮先排空
        if notifs:
            messages.append({"role": "user", "content": f"<background-results>..."})
        response = client.messages.create(...)  # ← 再调用 LLM
```

### 架构变更对比

| 组件 | s07 之前 | s08 之后 |
|------|----------|----------|
| 执行方式 | 仅阻塞 | 阻塞 + 后台线程 |
| 通知机制 | 无 | 每轮排空的队列 |
| 并发 | 无 | daemon 守护线程 |

### 后台任务 vs 子 Agent 的本质区别

| | 后台任务 (s08) | 子 Agent (s04) |
|--|---------------|----------------|
| 执行内容 | shell 命令（无思考） | LLM 推理 + 工具调用 |
| 有没有智能 | 没有，纯执行 | 有，能自主决策 |
| 上下文 | 无 | 独立 messages 列表 |
| 通信方式 | 完成后推通知队列 | 直接返回结果 |
| 适用场景 | 慢 I/O（编译、测试） | 需要智能的子任务 |

### 可泛化原则

**"不阻塞主流程，完成后通知"** 是异步系统的通用模式：
- 消息队列（Kafka/RabbitMQ）：生产者发消息立即返回，消费者异步处理
- 异步 HTTP（async/await）：发请求不等响应，回调处理结果
- 操作系统 I/O 中断：CPU 不等磁盘，I/O 完成后中断通知

类比单核 CPU 调度：主线程 = CPU，后台线程 = I/O 中断处理器，通知队列 = 中断向量表。

---

## 二、学习过程反思

### 答对的地方

- 正确判断同步 Agent 无法并行处理慢操作
- 正确联想到 CPU 单核调度和 I/O 并行化的类比
- 正确理解 LLM 调用前消费队列的时机选择原因
- 正确理解多线程写队列需要加锁（互斥）的必要性

### 犯的错误或思维漏洞

- **超时的影响**：最初认为"主 Agent 会一直在等"，但其实后台线程是异步的，主循环不会等。正确理解是：无超时会导致该线程变僵尸线程，通知队列永远没有结果，依赖该任务的后续任务永远被阻塞，且资源泄漏。

### 思考不够清晰的地方

- daemon 线程和普通线程的区别需要加深记忆（主进程退出时 daemon 被强制杀死，普通线程让主进程等待）

---

## 三、教学引导记录

### 产生顿悟的类比/问题

- "循环保持单线程，只有子进程 I/O 被并行化"——单核 CPU + I/O 中断的类比让整个架构一下子清晰了，Agent 的异步模型和操作系统调度是同构的

- 后台任务 vs 子 Agent 的对比：前者无智能纯执行，后者有 LLM 推理——澄清了两个看似相似机制的本质边界

### 需要后续关注的跨课程线索

- s09 多 Agent 团队中，多个 Agent 如何协调任务图里的任务（结合 s07 的 owner 字段）
- 后台任务的通知队列和 s09 的消息总线（MessageBus）有何关系

---

*学习日期：2026-03-27*
*提交 commit：待提交*
