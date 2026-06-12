---
name: fleet
description: AI-first 部署到 fleet panel(从代码到上线 HTTPS,workflow 驱动)
version: 0.0.1
---

# Fleet Skill

帮 AI IDE agent(Claude Code / Cursor / Codex)驱动 fleet panel 完成
部署、扩缩容、回滚、排障等全流程。设计文档:
[`fleetpanel/docs/vibe-deploy.md`](https://git.puzizi.cn/launchpad/fleetpanel/-/blob/main/docs/vibe-deploy.md)。

## 何时套用

- 用户说「帮我发到 fleet / 上线 / 部署 / 滚动 / 回滚 / push 上去」
- 用户在 `.fleet/state.yaml` 存在的项目里提到「域名 / HTTPS / 扩 / 缩」
- 用户问「上次 push 是不是挂了 / 为什么没切过去 / 容器起不来」

## 前提 0:fleet CLI 已安装

进入任何 workflow 前先 `command -v fleet` 确认 CLI 在;不在就装:

```sh
curl -fsSL https://git.puzizi.cn/launchpad/vibe-coding-client/-/raw/main/install.sh | sh
```

(可加版本参数钉版本:`… | sh -s -- v0.12.0`;私有实例需 `GITLAB_TOKEN`。)
版本用 `fleet --version` 看 —— 是 flag,**没有** `fleet version` 子命令。
装好后没登录过(`~/.fleet/credentials` 不存在)→ 套
[`panel-bootstrap`](workflows/panel-bootstrap.md):没面板帮装(没服务器连
机器一起买),有面板直接 `fleet auth login`。

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
| `panel-bootstrap`               | 冷启动:还没有面板 —— 买机装面板 / 已有服务器装面板 / 已有面板登录 |
| `host-enroll`                   | 加新机器;先问有没有服务器(没有→帮买),再按云厂商分支 |
| `host-buy-aliyun`               | 没有服务器:对话内选配 / 询价 / AutoPay=false 下单 / 扫码支付 / UserData 自动接入 |
| `host-enroll-aliyun`            | 阿里云 ECS:本机 `aliyun` CLI 查实例 / 开安全组 / 云助手接管 |
| `host-enroll-tencent`           | 腾讯云 CVM:本机 `tccli` 查实例 / 开安全组 / TAT 接管 |
| `source-pick`                   | 确定项目 source 类型;preflight 报 source gap |
| `expose-domain`                 | 补齐 expose.domain;preflight 报 cert gap(域名未设)|
| `rollback`                      | 回滚一个 instance                            |

## 编排原则

详见 [`conventions.md`](conventions.md):全局 `--output json` / exit 协议 /
inline 二选一 / `user_actions_required` / 资源锁退避 / 镜像版本号决策(D20)/
init 时多 host 选择(D21)。

## CLI 速查

详见 [`commands.md`](commands.md)。
