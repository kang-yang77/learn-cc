# Context Compact

> *"上下文总会满, 要有办法腾地方"* — 四层压缩策略, 便宜的先跑贵的后跑。
>
> **Harness 层**: 压缩 — 干净的记忆, 无限的会话。

## 背景

Agent 持续读文件、执行命令时，消息与工具结果不断累积。上下文耗尽会触发 `prompt_too_long`；过期内容也会分散注意力并增加成本。因此要保留任务目标与近期状态，逐步移除低价值信息。

## 解决办法

每次请求模型前依次执行四层压缩：

1. `tool_result_budget`：末批结果超过 200000 字符时，将大于 30000 字符的结果落盘，只留路径与预览。
2. `snip_compact`：超过 50 条消息时保留头部 3 条与近期消息，裁掉中段。
3. `micro_compact`：仅保留最近 3 个工具结果全文，将更早的大结果替换为占位符。
4. `compact_history`：仍超限时保存 transcript，让 LLM 将历史总结成一条消息。

```python
while True:
    messages[:] = tool_result_budget(messages)
    messages[:] = snip_compact(messages)
    messages[:] = micro_compact(messages)

    if estimate_size(messages) > CONTEXT_LIMIT:
        messages[:] = compact_history(messages)

    try:
        response = client.messages.create(...)
    except Exception as error:
        if is_prompt_too_long(error) and reactive_retries < 1:
            messages[:] = reactive_compact(messages)
            reactive_retries += 1
            continue
        raise
```

`reactive_compact` 是超限后的应急路径：总结旧历史、保留最近约 5 条消息，再重试一次。

## 原理

压缩按成本递增：前三层不调用模型，全量摘要需要额外请求，最后执行。实际先运行编号为 L3 的 budget，因为 micro 若先替换旧结果，原文就无法落盘。

裁剪不能拆开 `assistant(tool_use)` 与 `user(tool_result)`，否则协议失效。摘要须保留目标、决定、已改文件、剩余工作和约束；transcript 仅用于追溯。

当前 `estimate_size()` 统计字符而非 token。`write_transcript()` 使用未导入的 `time`，会使摘要失败；循环虽处理 `compact` 调用，工具列表却未注册它，模型无法主动触发。压缩有损，错误摘要会影响后续判断。

## Claude Code 生产级别实现

Claude Code 依次执行结果预算、历史裁剪、micro compact、`contextCollapse`、auto compact。它使用精确 token 预算，阈值为窗口减去最大输出和约 13000 token 缓冲区。

旧工具结果可按时间或缓存编辑压缩。`readFileState` 让未变化文件返回简短标记；压缩后按预算恢复近期文件、计划及 Agent、Skill、Tool 上下文。

摘要提示禁止工具调用，并要求输出结构化总结。系统为连续失败设置熔断上限，按完整消息组回退，避免孤立结果。`contextCollapse` 由独立提交与阻塞流程管理；手动压缩和应急 fallback 仍是独立路径。
