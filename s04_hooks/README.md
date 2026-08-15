# Hooks

## 背景

Agent 循环需要加入权限、日志、结果校验和退出清理。如果都写进循环，每次扩展都要修改核心流程，容易造成职责混杂并破坏工具调用协议。

Hooks 的目标是在生命周期中预留稳定扩展点：循环只负责触发事件，具体功能由外部回调完成。

## 解决办法

使用注册表保存“事件 -> 回调列表”，通过 `register_hook()` 注册，通过 `trigger_hooks()` 按顺序执行：

```python
HOOKS = {
    "UserPromptSubmit": [],
    "PreToolUse": [],
    "PostToolUse": [],
    "Stop": [],
}

def register_hook(event, callback):
    HOOKS[event].append(callback)

def trigger_hooks(event, *args):
    for callback in HOOKS[event]:
        result = callback(*args)
        if result is not None:
            return result
    return None
```

四个事件对应输入提交、工具执行前后和循环停止前。权限与日志注册到 `PreToolUse`，输出检查注册到 `PostToolUse`。

当前循环只接入了 `PreToolUse`：

```python
blocked = trigger_hooks("PreToolUse", block)
if blocked:
    results.append({
        "type": "tool_result",
        "tool_use_id": block.id,
        "content": str(blocked),
    })
    continue

output = TOOL_HANDLERS[block.name](**block.input)
```

## 原理

Hook 将稳定流程与可变策略解耦：事件定义“何时扩展”，回调定义“做什么”，注册表连接二者。

回调按注册顺序执行。`None` 表示继续，非 `None` 表示短路，因此权限 Hook 能在副作用前阻止工具。注册不会执行回调，每个事件仍需对应的 `trigger_hooks()` 调用点。

## 不足

- `UserPromptSubmit`、`PostToolUse` 和 `Stop` 虽已注册，但当前没有触发点，因此尚未生效。
- 权限 Hook 短路后，后续日志 Hook 不会运行，审计可能缺失。
- 返回值混合了阻止、消息和控制流语义。
- 没有异常隔离、超时、异步、匹配器和递归保护。

## Claude Code 生产级别实现

Claude Code 的 Hooks 覆盖工具、会话、权限、子 Agent、上下文压缩和团队协作等生命周期，包括 `PreToolUse`、`PermissionRequest`、`SubagentStop`、`PreCompact` 等事件。

结构化 `HookResult` 可携带执行结果、阻塞错误、权限决定、更新后的输入、附加上下文和停止标记，明确区分各种控制行为。

Hook 返回 `allow` 仍不能绕过配置中的 `deny` 或 `ask`；Stop Hook 使用活动状态避免反复阻止停止；PostToolUse 可通过 `preventContinuation` 正常结束 Agent，而非依靠异常退出。
