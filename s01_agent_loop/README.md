# Agent Loop

## 背景

普通模型只能生成文本。即使它给出了正确的 Shell 命令，也不会自动执行，更无法观察结果后继续判断。用户必须反复复制命令、执行并粘贴输出，实际承担了模型与计算机之间的调度工作。

Agent 的关键不是一次回答更聪明，而是建立“推理、行动、观察、再推理”的闭环，让模型能够根据真实执行结果持续推进任务。

## 解决办法

向模型声明一个 Bash 工具，用 `while True` 驱动对话。模型请求工具时执行命令，将结果作为 `tool_result` 追加到历史，再次调用模型；模型不再请求工具时退出。

```python
def agent_loop(messages):
    while True:
        response = client.messages.create(
            model=MODEL,
            system=SYSTEM,
            messages=messages,
            tools=TOOLS,
            max_tokens=8000,
        )
        messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason != "tool_use":
            return

        results = []
        for block in response.content:
            if block.type == "tool_use":
                output = run_bash(block.input["command"])
                results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": output,
                })
        messages.append({"role": "user", "content": results})
```

`run_bash()` 设置 120 秒超时，合并标准输出和错误输出，并限制返回长度，避免单次命令无限占用循环。

## 原理

循环实现了 ReAct 模式：模型负责 Reason 和 Act，宿主程序执行动作并返回 Observation。`tool_use_id` 将每个结果与原调用关联；模型一次生成多个调用时，也必须逐一返回结果。模型响应要先写入历史，再追加工具结果，否则消息链缺少产生调用的上下文。

`stop_reason` 是当前实现的状态开关：等于 `tool_use` 就继续，否则结束。外层 `history` 保留多轮会话，使后续问题能复用此前信息。

当前实现使用 `shell=True`，关键词拦截不能构成可靠沙箱；同时缺少最大轮次、取消信号、流式处理、上下文压缩和错误恢复，模型若持续调用工具可能长期循环。

## Claude Code 生产级别实现

Claude Code 的核心仍是 Agent Loop，但不会只依赖最终的 `stop_reason`。流式响应中一旦发现 `tool_use` 内容块，就设置继续标志，因为结束原因可能尚未到达或不可靠。

生产循环维护更完整的状态，包括消息、工具上下文、轮次、压缩状态、输出 token 恢复次数、Stop Hook 状态和上次转换原因。退出路径也覆盖最大轮次、用户中止、上下文过长、模型错误和预算耗尽，并为部分错误提供重试或压缩恢复。

`StreamingToolExecutor` 可在模型仍生成响应时启动工具，并根据并发安全性决定并行或独占执行。复杂机制最终都服务于同一个闭环：识别工具请求、可靠执行、返回观察结果，再决定继续或停止。
