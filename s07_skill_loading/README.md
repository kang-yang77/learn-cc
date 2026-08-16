# Skill Loading

> *"用到时再加载, 别全塞 prompt 里"* — 通过 tool_result 注入, 不塞 system prompt。
>
> **Harness 层**: 知识 — 按需加载, 不堆满上下文。

## 背景

Agent 可能需要代码审查、PDF 或 MCP 等专业知识。若全部写进 system prompt，每次调用都会携带无关内容，浪费上下文并削弱任务重点。

理想方式是让模型始终知道“有哪些知识可用”，但只在任务相关时加载完整内容，即渐进式披露。

## 解决办法

每个 Skill 放在独立目录中，以 `SKILL.md` 作为入口。启动时扫描 `skills/`，解析 YAML frontmatter 中的 `name` 和 `description`，建立注册表：

```python
def _scan_skills():
    for directory in sorted(SKILLS_DIR.iterdir()):
        manifest = directory / "SKILL.md"
        if not manifest.exists():
            continue

        raw = manifest.read_text()
        meta, _ = _parse_frontmatter(raw)
        name = meta.get("name", directory.name)
        SKILL_REGISTRY[name] = {
            "name": name,
            "description": meta.get("description", ""),
            "content": raw,
        }
```

系统提示只注入名称和描述。模型判断某项技能与任务相关时，再调用 `load_skill` 获取完整 `SKILL.md`：

```python
def load_skill(name: str) -> str:
    skill = SKILL_REGISTRY.get(name)
    if not skill:
        return f"Skill not found: {name}"
    return skill["content"]
```

注册表按名称查询，不把模型输入拼成路径，可避免路径遍历。Skill 作为 `tool_result` 进入历史，并可指引现有工具读取其他资源。

## 原理

Skill 采用两级加载：目录是低成本索引，常驻 system prompt；正文是高成本知识，只在需要时进入上下文。模型先根据描述完成检索，再显式加载正文，效果类似按需导入模块。

注册表只在启动时构建，修改 Skill 不会自动刷新，同名条目会相互覆盖。当前仅识别直接子目录，只使用名称和描述。正文加载后会持续占用上下文，直至会话结束或压缩。

子 Agent 没有 `load_skill`。目录很大时，所有描述仍会扩大 system prompt，需要目录预算和筛选策略。

## Claude Code 生产级别实现

Claude Code 从企业策略、用户、项目、`--add-dir`、插件、内置、动态和 MCP 等来源发现并合并 Skill。

Frontmatter 除名称和描述外，还可控制适用条件、`allowed-tools`、`context`、模型、Hooks、路径匹配和用户是否可直接调用。`context: inline` 在当前对话展开，`context: fork` 可交给独立子 Agent 执行。

生产实现同样分为目录与正文两级。目录预算约占上下文窗口的 1%，并有字符上限。Skill 工具接收名称和参数；界面只显示启动信息，正文通过新消息注入。

因此生产系统不仅负责读取 Markdown，还处理多来源发现、元数据校验、条件激活、上下文预算、权限范围和 inline/fork 执行方式。
