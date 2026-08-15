# Subagent

## 背景

复杂任务常包含搜索、调研或局部实现等子问题。若全部放在主对话中，大量文件内容和工具结果会挤占上下文，使主 Agent 忘记目标或被局部细节带偏。

子 Agent 的作用是把一个边界明确的子问题放进独立上下文执行，只把结论交还父 Agent。主对话保留任务主线，子过程可以在完成后丢弃。

## 解决办法

为父 Agent 增加 `task` 工具，其 handler 是 `spawn_subagent()`。函数使用任务描述创建全新的 `messages`，配合独立的 `SUB_SYSTEM` 运行另一套 Agent Loop，最多执行 30 轮。

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

子 Agent 只有 Bash 和文件类工具，没有 `task` 与 `todo_write`，因此不能递归委派。它的工具调用仍经过 `PreToolUse` 和 `PostToolUse` Hooks。执行结束后只返回文本摘要，父 Agent 将它当作普通 `tool_result` 继续处理。

## 原理

父子 Agent 拥有不同的消息列表，因此子 Agent 的搜索过程不会进入父上下文；但二者共享模型客户端、工作目录和工具实现。子 Agent 修改的文件对父 Agent 立即可见，所以这是**上下文隔离**，不是进程、权限或文件系统隔离。

只回传摘要能显著减少上下文占用，但也会损失来源和中间证据，因此任务描述应包含明确范围、期望输出和完成标准。移除 `task` 工具形成简单的递归保护，30 轮上限防止子循环无限运行。

当前实现是同步的，父 Agent 会等待子 Agent 完成；没有并发、取消、进度通知和独立资源限额。权限 Hook 被复用，但两者共享终端输入，子 Agent 的审批会阻塞父流程。

## Claude Code 生产级别实现

Claude Code 支持普通 Subagent、General-Purpose Agent 和 Fork Subagent。普通模式使用新上下文；Fork 模式构造与父对话兼容的消息前缀，使 system prompt、tools、model、thinking 配置和消息前缀保持一致，从而复用 Prompt Cache。

子 Agent 上下文并非完全独立：它拥有新的调用链标识和子级取消控制器，父级取消会向下传播，部分文件读取状态可从父级复制。递归委派通过工具禁用集合、上下文标记和深度信息共同限制，而不只依赖提示词。

子 Agent 的权限模式可设为 `bubble`，将审批请求冒泡到父终端。除同步等待外，Claude Code 还支持后台执行：父 Agent 先收到启动结果，子任务完成后再通过通知回传。生产实现因此同时处理缓存复用、权限继承、取消传播、递归控制和异步生命周期。
