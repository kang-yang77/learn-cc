# Subagent

> *"大任务拆小, 每个小任务干净的上下文"* — Subagent 用独立 messages[], 不污染主对话。
>
> **Harness 层**: 子 Agent — 上下文隔离, 注意力不漂移。

## 背景

复杂任务中的搜索和调研会产生大量中间结果，挤占主对话，使 Agent 忘记目标。

子 Agent 在独立上下文处理明确的子问题，只把结论交还父 Agent，完成后丢弃过程。

## 解决办法

为父 Agent 增加 `task` 工具。`spawn_subagent()` 创建新的 `messages`，使用独立 `SUB_SYSTEM` 运行最多 30 轮。

```python
def spawn_subagent(description: str) -> str:
    messages = [{"role": "user", "content": description}]

    for _ in range(30):
        response = client.messages.create(
            model=MODEL,
            system=SUB_SYSTEM,
            messages=messages,
            tools=SUB_TOOLS,
            max_tokens=8000,
        )
        messages.append({"role": "assistant", "content": response.content})
        if response.stop_reason != "tool_use":
            break

        results = []
        for block in response.content:
            if block.type != "tool_use":
                continue
            blocked = trigger_hooks("PreToolUse", block)
            if blocked:
                output = str(blocked)
            else:
                output = SUB_HANDLERS[block.name](**block.input)
                trigger_hooks("PostToolUse", block, output)
            results.append({
                "type": "tool_result",
                "tool_use_id": block.id,
                "content": output,
            })
        messages.append({"role": "user", "content": results})

    return extract_text(messages[-1]["content"])
```

子 Agent 只有 Bash 和文件工具，没有 `task` 与 `todo_write`，不能递归委派。调用仍经过权限 Hooks，结束后只向父 Agent 返回文本摘要。

## 原理

父子 Agent 的消息列表不同，但共享模型、工作目录和工具。文件修改立即互相可见，所以这是**上下文隔离**，不是资源隔离。

摘要能减少上下文，也会损失中间证据，因此任务描述应包含范围和完成标准。移除 `task` 防止递归，30 轮上限避免无限运行。

当前为同步执行，没有并发、取消、进度通知和独立资源限额；子 Agent 的审批会阻塞父流程。

## Claude Code 生产级别实现

Claude Code 支持普通、General-Purpose 和 Fork Subagent。Fork 模式保持 system prompt、tools、model、thinking 配置及消息前缀一致，以复用 Prompt Cache。

子 Agent 上下文并非完全独立：它拥有新的调用链标识和子级取消控制器，父级取消会向下传播，部分文件读取状态可从父级复制。递归委派通过工具禁用集合、上下文标记和深度信息共同限制，而不只依赖提示词。

子 Agent 的权限模式可设为 `bubble`，将审批请求冒泡到父终端。除同步等待外，Claude Code 还支持后台执行：父 Agent 先收到启动结果，子任务完成后再通过通知回传。生产实现因此同时处理缓存复用、权限继承、取消传播、递归控制和异步生命周期。
