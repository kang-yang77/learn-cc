# 权限管理（Permission）

## 背景

工具调用让模型能够读写文件和执行 Shell 命令，但 `tool_use` 只是一份执行意图。模型可能误解需求、生成错误命令或受到提示注入影响，系统提示词不能充当安全边界。

因此，工具执行前必须由宿主程序作出最终决定：低风险操作直接允许，依赖上下文的操作询问用户，高危操作直接拒绝。

## 解决办法

在工具分发器前加入 `check_permission()`，让每次调用依次经过三道闸门：

```text
tool_use
   ↓
硬拒绝列表 ──命中──> deny
   ↓ 未命中
上下文规则 ──命中──> 用户确认 ──拒绝──> deny
   ↓ 未命中             ↓ 允许
   └──────────────────> 执行工具
```

硬拒绝列表拦截高危命令；上下文规则识别越界路径、删除和敏感写入；用户确认仅接受 `y` 或 `yes`，其他输入默认拒绝。

```python
if not check_permission(block):
    results.append({
        "type": "tool_result",
        "tool_use_id": block.id,
        "content": "Permission denied.",
    })
    continue

output = TOOL_HANDLERS[block.name](**block.input)
```

被拒绝的调用仍需返回对应的 `tool_result`，使模型知道操作未执行，并保持协议完整。

## 原理

权限检查位于所有工具共享的执行入口。规则按“硬拒绝、人工确认、默认允许”处理，避免宽泛授权覆盖具体限制。路径先通过 `resolve()` 规范化，可识别 `..` 和符号链接造成的越界访问；审批采用 fail-closed 策略，输入不明确时拒绝。

权限、参数校验和执行隔离是不同防线。用户允许不能突破 `safe_path()`、操作系统权限或沙箱。当前 Bash 规则仅做字符串匹配，可能被引号、变量和 Shell 展开绕过，并非可靠的命令沙箱。

## Claude Code 生产级别实现

Claude Code 内部使用 `allow`、`deny`、`ask` 和 `passthrough` 四种结果。`passthrough` 表示当前工具不作决定，继续交给通用权限管线。

工具执行会经过 schema 与语义校验、`PreToolUse` Hooks、配置规则、工具权限判断和用户审批，许可后才执行。规则可来自用户、项目、本地、企业策略、CLI 和会话授权。

不同工具拥有各自的权限语义：文件工具检查路径，Bash 分析命令风险，MCP 工具可提供破坏性注解。自动模式结合白名单、权限模式和分类器减少询问，无法安全判断时回退到人工审批。子 Agent 的请求可通过 `bubble` 冒泡到父 Agent。权限决定是否授权，沙箱限制实际能力。
