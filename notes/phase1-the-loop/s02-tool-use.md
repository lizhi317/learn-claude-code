# s02 - Tool Use

> **核心问题**: 怎样让 Agent 拥有更多能力而不修改核心循环？
> **参考文档**: `docs/zh/s02-tool-use.md`
> **参考实现**: `agents/s02_tool_use.py`
> **状态**: ✅ 已完成
> **学习日期**: 2026-03-21

---

## 一、知识总结

### 本节本质（一句话）

> **加工具 = 加 handler + 加 schema，循环永远不变。**

### 关键设计决策

| 设计 | 原因 |
|------|------|
| dispatch map（字典路由） | 对扩展开放，对修改封闭——加工具不动循环 |
| `safe_path`: 先 `resolve()` 再 `is_relative_to()` | 展开所有 `..` 后再检查真实落点，防路径逃逸 |
| 找不到工具返回字符串而非抛异常 | 错误本身是工具结果，模型可自行调整策略，循环不崩 |

### 架构变更（s01 → s02）

| 组件 | s01 | s02 |
|------|-----|-----|
| 工具数量 | 1（仅 bash） | 4（bash/read/write/edit） |
| 工具分发 | 硬编码 `run_bash()` | `TOOL_HANDLERS` 字典查找 |
| 路径安全 | 无 | `safe_path()` 沙箱 |
| 循环本体 | — | 一行未动 |

### 关键代码

```python
# dispatch map：工具名 → 处理函数
TOOL_HANDLERS = {
    "bash":       lambda **kw: run_bash(kw["command"]),
    "read_file":  lambda **kw: run_read(kw["path"], kw.get("limit")),
    "write_file": lambda **kw: run_write(kw["path"], kw["content"]),
    "edit_file":  lambda **kw: run_edit(kw["path"], kw["old_text"], kw["new_text"]),
}

# 循环中一行查字典，找不到返回字符串（不抛异常）
handler = TOOL_HANDLERS.get(block.name)
output = handler(**block.input) if handler else f"Unknown tool: {block.name}"
```

```python
# safe_path：先展开，再检查
def safe_path(p: str) -> Path:
    path = (WORKDIR / p).resolve()        # 展开所有 ..
    if not path.is_relative_to(WORKDIR):  # 检查真实落点
        raise ValueError(f"Path escapes workspace: {p}")
    return path
```

### 可泛化原则

- **dispatch map = Spring 路由表**，本质是同一个设计：把"名称"映射到"处理逻辑"，对扩展开放，对修改封闭
- **能转成数据的错误就不要变成异常**，适用于任何需要保持流程存活的系统

---

## 二、学习过程反思

### 答对的地方

- 识别出硬编码 if-elif 的可维护性问题，主动想到"配置文件"方向
- 理解了路径限定的目标，提出用父目录层次判断（方向正确）
- 正确理解了返回字符串 vs 抛异常对 loop 存活的影响，并说到了本质

### 思维漏洞

**路径安全**：最初想到"数 `../` 层数"——这种方式在路径混合正常目录名和 `..` 时容易出错。
- 错误思路：数 `../` 的数量判断层数
- 正确做法：`resolve()` 先把所有 `..` 展开成绝对路径，再用 `is_relative_to()` 检查真实落点

### 模糊地带

- `pathlib` 常用方法不熟练（`resolve()`、`is_relative_to()`），可课后补一下

---

## 三、教学引导记录

### 产生顿悟的类比

**Spring `@RequestMapping` → dispatch map**：把抽象的字典路由和已有的 Java 经验直接连接，瞬间理解"为什么字典比 if-elif 好"。

### 跨课程线索

| 问题 | 解决课程 |
|------|----------|
| "能转成数据的错误不要变成异常"这个原则 | s08 后台任务中会再次出现 |
