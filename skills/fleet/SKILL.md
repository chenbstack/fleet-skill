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
- 用户在已登记的 fleet 项目目录里(`fleet projects which` 能解析到)提到「域名 / HTTPS / 扩 / 缩」
- 用户问「上次 push 是不是挂了 / 为什么没切过去 / 容器起不来」

## 前提 0:fleet CLI 已安装

进入任何 workflow 前先确认 CLI 在(macOS/Linux `command -v fleet`,Windows
`where fleet`);不在就按平台装(官网 https://fleet.puzizi.cn 分发):

```sh
# macOS / Linux
curl -fsSL https://fleet.puzizi.cn/install-cli.sh | sh
```

```powershell
# Windows PowerShell
irm https://fleet.puzizi.cn/install-cli.ps1 | iex
```

```bat
:: Windows cmd.exe(借道 PowerShell)
powershell -ExecutionPolicy Bypass -Command "irm https://fleet.puzizi.cn/install-cli.ps1 | iex"
```

(钉版本:`.sh` 加 `… | sh -s -- v0.20.1`;`.ps1` 用
`& ([scriptblock]::Create((irm https://fleet.puzizi.cn/install-cli.ps1))) v0.20.1`。)
版本用 `fleet --version` 看 —— 是 flag,**没有** `fleet version` 子命令。
升级用 `fleet update`(自更新到官网最新版,下载 + sha256 + 原子替换);CLI 每次
启动也会后台静默查新版,有新版时命令末尾提示一行(网络失败静默,不影响操作)。
装好后没登录过(`~/.fleet/credentials` 不存在)→ 套
[`panel-bootstrap`](workflows/panel-bootstrap.md):没面板帮装(没服务器连
机器一起买),有面板直接 `fleet auth login`。

用户手里**已有面板签发的 PAT**(`fpat_…` 开头,通常来自面板「访问凭证」页
的交付文本)→ 不走 bootstrap,直接验证并落盘:

```sh
fleet auth login --panel "http://<host>:<port>/<入口路径>" --with-token "fpat_…"
fleet auth whoami --output json   # 确认
```

`--panel` 必须是**含安全入口的完整地址**(用户浏览器地址栏端口后面那段
路径)。缺入口前缀会吃到 entrance 防护的 fake 404,典型症状就是下面的
`panel.no_cliapi` 误报。

## 前提:EE panel

vibe-deploy 的服务端 API(`/api/v1/cli`)**仅 EE 版 panel 提供**。命令返回
reason `panel.no_cliapi` 时,有两种可能,**先排查第一种再下结论**:

1. **panel URL 缺安全入口前缀** —— entrance 防护对未带前缀的路径返回伪装
   404,CLI 会误读成「不认识 CLI API」。先核对 URL,
   `curl -fsS "http://<host>:<port>/<入口路径>/healthz"` 通了再试。
2. 真的是 CE panel —— 告知用户:「这台 panel 是 CE(开源版),vibe deploy
   需要 fleetpanel-ee;CE 请走 panel UI 操作」。不要重试,不要试图用别的
   命令绕。

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

详见 [`conventions.md`](conventions.md):项目解析(注册表 + 仓内 state.yaml)/
全局 `--output json` / exit 协议 / inline 二选一 / `user_actions_required` /
资源锁退避 / 镜像版本号决策(D20)/ init 时多 host 选择(D21)。

## CLI 速查

详见 [`commands.md`](commands.md)。
