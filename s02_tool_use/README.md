# Tool Use

> *"加一个工具, 只加一个 handler"* — 循环不用动, 新工具注册进 dispatch map 就行。
>
> **Harness 层**: 工具分发 — 扩展模型能触达的边界。

## 背景

只有 Bash 时，读文件、修改文本和查找路径都需要模型拼接 Shell 命令。模型必须把“读取文件”先翻译为 `cat`，不仅浪费 token，还会引入转义、平台差异和命令注入风险。

工具应该直接表达领域动作，并通过结构化参数约束输入。这样模型选择的是“做什么”，宿主程序负责“怎么做”。

## 解决办法

定义 `bash`、`read_file`、`write_file`、`edit_file` 和 `glob` 五个工具，并使用分发表建立工具名与处理函数的映射：

```python
TOOL_HANDLERS = {
    "bash": run_bash,
    "read_file": run_read,
    "write_file": run_write,
    "edit_file": run_edit,
    "glob": run_glob,
}

for block in response.content:
    if block.type == "tool_use":
        handler = TOOL_HANDLERS.get(block.name)
        output = handler(**block.input) if handler else f"Unknown: {block.name}"
```

增加工具只需声明名称、描述和 `input_schema`，再注册对应处理函数，Agent Loop 无需感知具体工具。

文件工具统一通过 `safe_path()` 解析路径，拒绝工作区之外的目标：

```python
def safe_path(path: str) -> Path:
    resolved = (WORKDIR / path).resolve()
    if not resolved.is_relative_to(WORKDIR):
        raise ValueError(f"Path escapes workspace: {path}")
    return resolved
```

## 原理

工具定义是模型与宿主程序之间的契约：名称和描述帮助模型选工具，JSON Schema 规定参数结构，handler 承担真实副作用。分发表把协议层与执行层解耦，避免在循环中堆积条件分支。

模型可在一次响应中产生多个 `tool_use`。当前实现按内容顺序串行执行，因此结果稳定，但互不依赖的读取也无法并行。`read_file` 支持行数限制，`edit_file` 只替换第一个精确匹配，`glob` 和文件工具限制在工作区内；Bash 仍能通过 Shell 访问更大范围，不受 `safe_path()` 约束。

JSON Schema 主要约束模型生成参数，handler 仍需处理文件不存在、参数语义错误和操作系统异常，不能把 schema 当作完整安全校验。

## Claude Code 生产级别实现

Claude Code 将每个工具封装为独立对象，集中声明 schema、输入验证、权限检查、并发属性和执行逻辑。调用会依次经过结构校验、工具级 `validateInput()`、`PreToolUse` Hooks、权限决策，最后才执行。

并发不是简单按“读或写”分类，而由工具结合具体输入判断。调度器保持原始顺序，将连续的并发安全调用组成批次并行执行，遇到有副作用的调用则单独串行；不同批次仍严格有序。

流式执行器可以在模型尚未结束输出时启动已完整生成的工具调用。工具结果还具有大小限制，超限内容可落盘，只向模型返回预览和路径，避免大输出迅速耗尽上下文。生产工具系统因此同时处理契约、验证、权限、调度和结果生命周期。
