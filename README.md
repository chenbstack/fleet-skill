# fleet-skill

给 AI IDE agent(Claude Code / Cursor / Codex)用的 fleet 部署 SKILL。
按 [`vibe-deploy.md` D17](../fleetpanel/docs/vibe-deploy.md) 分发:
GitHub repo + Anthropic marketplace 双路。

## 安装

```sh
# Claude Code
/skill install fleet
```

或 clone 后链接到 `~/.claude/skills/fleet`。

## 结构

```
SKILL.md          # skill 入口 / 何时套用 / workflow 索引
commands.md       # CLI 命令速查
conventions.md    # 编排约定(exit / inline / 版本号 / host 选择)
workflows/        # 一文件一 workflow,YAML frontmatter + 步骤
```

## 设计来源

设计文档:[`fleetpanel/docs/vibe-deploy.md`](../fleetpanel/docs/vibe-deploy.md)。
预览 HTML:[`fleet-vibe-deploy-design.html`](../fleet-vibe-deploy-design.html)、
[`fleet-vibe-deploy-demo.html`](../fleet-vibe-deploy-demo.html)。
