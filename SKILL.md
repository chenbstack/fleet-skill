---
name: fleet
description: AI-first 部署到 fleet panel(从代码到上线 HTTPS,workflow 驱动)
version: 0.0.1
---

# Fleet Skill

帮 AI IDE agent(Claude Code / Cursor / Codex)驱动 fleet panel 完成
部署、扩缩容、回滚、排障等全流程。设计文档:
[`fleetpanel/docs/vibe-deploy.md`](https://github.com/fleetpanel/fleetpanel/blob/main/docs/vibe-deploy.md)。

## 何时套用

- 用户说「帮我发到 fleet / 上线 / 部署 / 滚动 / 回滚 / push 上去」
- 用户在 `.fleet/state.yaml` 存在的项目里提到「域名 / HTTPS / 扩 / 缩」
- 用户问「上次 push 是不是挂了 / 为什么没切过去 / 容器起不来」

## 前提:EE panel

vibe-deploy 的服务端 API(`/api/v1/cli`)**仅 EE 版 panel 提供**。任何命令
返回 reason `panel.no_cliapi` 时,说明对端是 CE panel —— 直接告知用户:
「这台 panel 是 CE(开源版),vibe deploy 需要 fleetpanel-ee;CE 请走 panel
UI 操作」。不要重试,不要试图用别的命令绕。

## Workflows

| id                              | 何时套用                                     |
| ------------------------------- | -------------------------------------------- |
| `app-publish`                   | 主线:从代码到 push 上线                     |
| `cert-enable-https`             | 给一个域名签发 + 装证书                      |
| `dns-account-add`               | 接入 DNS 服务商(支线)                      |
| `acme-account-add`              | 注册 ACME 账户(支线)                       |
| `ci-wire-github`                | 把 fleet push 织进 GitHub Actions            |
| `troubleshoot-build-failed`     | 编译挂了的诊断回路                           |
| `troubleshoot-runtime-failed`   | 容器跑挂了 / 起不来                          |
| `host-pick`                     | 选定/修正 state.hosts;preflight 报 host / drift gap |
| `host-enroll`                   | 加新机器                                     |
| `source-pick`                   | 确定项目 source 类型;preflight 报 source gap |
| `expose-domain`                 | 补齐 expose.domain;preflight 报 cert gap(域名未设)|
| `rollback`                      | 回滚一个 instance                            |

## 编排原则

详见 [`conventions.md`](conventions.md):全局 `--output json` / exit 协议 /
inline 二选一 / `user_actions_required` / 资源锁退避 / 镜像版本号决策(D20)/
init 时多 host 选择(D21)。

## CLI 速查

详见 [`commands.md`](commands.md)。
