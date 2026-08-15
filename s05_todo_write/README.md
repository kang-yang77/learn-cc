# TodoWrite

## 背景

Agent 处理多步任务时，容易被最新结果吸引并忘记初始目标，出现漏步、重复或提前结束。任务越长，单靠系统提示越难维持进度。

需要把隐含在模型上下文中的计划变成显式状态，让模型随时知道“还要做什么、正在做什么、已经完成什么”。

## 解决办法

增加 `todo_write` 工具，接收包含 `content` 和 `status` 的列表，状态只允许 `pending`、`in_progress` 和 `completed`。系统提示要求模型先创建计划，再持续更新。

```python
def run_todo_write(todos: list) -> str:
    global CURRENT_TODOS
    todos, error = _normalize_todos(todos)
    if error:
        return error

    CURRENT_TODOS = todos
    for todo in CURRENT_TODOS:
        print(todo["status"], todo["content"])
    return f"Updated {len(CURRENT_TODOS)} tasks"
```

该工具只记录计划，不执行任务。它复用普通工具的 schema、分发器和 `tool_result` 协议。

循环统计距离上次更新经过的工具轮次。连续三轮未调用 `todo_write` 时注入提醒，调用后计数归零。

```python
if rounds_since_todo >= 3 and messages:
    messages.append({
        "role": "user",
        "content": "<reminder>Update your todos.</reminder>",
    })
    rounds_since_todo = 0

if block.name == "todo_write":
    rounds_since_todo = 0
```

## 原理

TodoWrite 将计划外化为工作记忆。`pending -> in_progress -> completed` 构成状态机，使 Agent 确定范围、聚焦当前项并检查剩余任务。它不增加执行能力，只减少注意力漂移。

`_normalize_todos()` 检查列表、字段和状态枚举。每次调用会整体替换 `CURRENT_TODOS`，因此模型必须提交完整列表。

状态仅存于内存，退出即丢失；任务没有 ID、依赖和负责人，也未限制只能有一个 `in_progress`。固定轮次提醒不理解任务复杂度。字符串兼容分支依赖未导入的 `json` 和 `ast`，手工传入字符串会失败。

## Claude Code 生产级别实现

Claude Code 并存两套机制。TodoWrite V1 是内存列表，`activeForm` 用于在界面展示当前动作，适合单会话的轻量计划。

Task System V2 面向复杂和多 Agent 工作流：任务以文件持久化，`blockedBy` 表达依赖，文件锁保护并发更新，ownership 标识负责人。操作拆成 Create、Get、Update、List，并提供任务 Hooks。

Claude Code 没有固定“三轮未更新”规则。TodoWrite V1 会在多个 TODO 完成但缺少验证项时提示补充 verification。生产设计更关注状态恢复、依赖表达、并发协调和验证闭环。
