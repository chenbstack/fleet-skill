# fleet-skill

给 AI IDE agent(Claude Code / Cursor / Codex)用的 fleet 部署 SKILL。
按 [`vibe-deploy.md` D17](../fleetpanel/docs/vibe-deploy.md) 分发:
Claude Code plugin marketplace + 官网 URL 直装,双路。

## 安装

### Claude Code(推荐:plugin marketplace)

本仓就是一个 Claude Code marketplace(`fleet-vibe-deploy`),里面挂一个
插件 `fleet`。在 Claude Code 里跑两条:

```
/plugin marketplace add chenbstack/fleet-skill
/plugin install fleet@fleet-vibe-deploy
```

升级:`/plugin marketplace update fleet-vibe-deploy` 后再 `/plugin install`。

### 其它 agent(Cursor / Codex / 通用):官网 URL 直装

不走 Claude Code plugin 体系的 agent,按官网安装器把文件拉到技能目录:

```sh
curl -fsSL https://fleet.puzizi.cn/install-skill.sh | sh
```

或把这句话丢给你的 agent,让它自己读清单装:
> 阅读 https://fleet.puzizi.cn/skill-install.md 并按里面的清单安装这个技能。

### 手动 / 开发

clone 后把插件根软链到技能目录(注意大小写 `SKILL.md`):

```sh
ln -s "$PWD/skills/fleet" ~/.claude/skills/fleet
```

## 结构(Claude Code plugin 布局)

```
.claude-plugin/
  plugin.json        # 插件清单(name=fleet / version)
  marketplace.json   # marketplace 清单(name=fleet-vibe-deploy,挂 fleet 插件)
skills/fleet/
  SKILL.md           # skill 入口 / 何时套用 / workflow 索引
  commands.md        # CLI 命令速查
  conventions.md     # 编排约定(exit / inline / 版本号 / host 选择 / 项目解析)
  workflows/         # 一文件一 workflow,YAML frontmatter + 步骤
```

官网 `website/public/` 把 `skills/fleet/` 摊平成根级 `skill.md` / `commands.md`
/ `conventions.md` / `workflows/*.md` 直接 URL 分发(供上面的 URL 直装渠道
逐个拉取)。改 skill 内容认 `skills/fleet/` 为事实源,再同步到官网。

## 设计来源

设计文档:[`fleetpanel/docs/vibe-deploy.md`](../fleetpanel/docs/vibe-deploy.md)。
预览 HTML:[`fleet-vibe-deploy-design.html`](../fleet-vibe-deploy-design.html)、
[`fleet-vibe-deploy-demo.html`](../fleet-vibe-deploy-demo.html)。
