# CLI 命令速查

完整定义见 [`fleetpanel/docs/vibe-deploy.md §3`](https://github.com/fleetpanel/fleetpanel/blob/main/docs/vibe-deploy.md)。本表只列名字 + 一句话用途,供 AI 选命令时快速对应。

## 原子(一条 = 一个 panel REST)

| 命令族              | 子命令                                   |
| ------------------- | ---------------------------------------- |
| `auth`              | `login` / `logout` / `set-password` / `whoami` |
| `tokens`            | `list` / `create` / `revoke <id>`       |
| `users`             | `list` / `get <id>` / `create` / `patch <id>` / `delete <id>` / `reset-password <id>` —— panel 本地账号管理(RBAC policy 走 `patch`) |
| `hosts`             | `list` / `add` ⚠️ / `get <id>` / `remove <id>` —— `add` 暂走 panel UI(后端返 `host.enroll_via_ui`),CLI 入口保留但实际入网由 UI 完成 |
| `hosts components`  | `check <host>` / `install <host> --components docker,…`(docker · openresty · registry_pull · buildx 四件套) |
| `dns-accounts`      | `list` / `add` / `remove <id>`          |
| `acme-accounts`     | `list` / `add` / `remove <id>`          |
| `site-certs`        | `list` / `check <domain>` / `issue` / `renew <id>` / `remove <id>` / `deployments list\|add\|remove <cert-id>`(一证书 × N host)—— 无独立 `install`,`issue` 已含落地到 OpenResty |
| `image-registries`  | `list` / `add` / `remove <id>` / `events <id>` / `tags <id> <owner>/<name>` / `delete-tag <id> <owner>/<name>:<tag>` —— 后三个只对 builtin stashhub-lite 实例有效 |
| `app-instances`     | `list` / `create` / `get <id>` / `remove <id>` / `scale <id>` / `rollback <id>` —— 无 `update`,改 hosts/replicas 走 `scale` |
| `app-builds`        | `list` / `get <id>` / `retention [--set N \| --unset]` —— retention 不带 flag 时打印当前保留版本数 + 是否 default;`--set 0` 等于关掉 retention |
| `app-versions`      | `latest`(含 `latest_pushed_status`,failed 表示该版本号可同号重推;`active` 只在全部 rollout 成功后切换)/ `list` |
| `ci-events`         | `list` / `get <id>`                     |
| `exec-actions`      | `watch <id>`(别名 `stream`;exit code 反映 plan 终态)/ `list` / `get <id>` |
| `logs`              | `--build <id>` / `--tail` / `--level error` |
| `status`            | `[--metrics] [--wait <duration>]`       |
| `env`               | `list` / `set NAME=VALUE` / `remove NAME` |
| `init`              | `--app <name>` / `--host <label>`(可重复)/ `--source tarball\|dockerfile\|image\|static` / `--static-dir <dir>` / `--build panel\|ci\|external` / `--version-policy semver\|external\|calver` / `--domain` / `--tls` / `--panel <url>` / `--update`(只改显式给出的字段,resolve 剧本用它写回 state)/ `--reset`(整体重写,用户明说"重来"才加) |
| `push`              | `--version vX.Y.Z`(必填,见 D20)/ `--strategy rolling\|recreate\|canary`(默认 canary)/ `--canary-replicas` / `--canary-grace` / `--watch`(`-w`);服务端强制内嵌硬前置(D14)。**默认非阻塞**(gh 风格):202 受理即返回 `rollouts[]`,rollout 与版本终态推进全在服务端,断开无影响;**AI 优先 `--watch`**,一条命令流式跟完全部 rollout,exit code 反映成败。`state.source=static` 时 `--version` 不强制,自动 tar.gz `state.static_dir`(默认 `./dist`)上传到 `/apps/{app}/site:upload`,站点缺失时用 state 的 domain/host 自动创建(幂等);同样默认非阻塞、`--watch` 跟完 |
| `push status [app]` | 最近一次 push 的聚合状态:`{version, build_id, build_status, active, rollouts[]}`(每 rollout 含 host / status / action_id / error)。读持久化行,**panel 重启后仍可查**。`--wait` 阻塞到 build 终态并把终态映射进 exit code(finished=0 / failed=1),`--timeout` 默认 45m;不带 `--wait` 纯查询恒 exit 0 |
| `push watch [app]`  | attach 最近一次 push(`gh run watch` 等价):逐 rollout 重放 + 跟日志流,再等 build 终态;exit code 反映成败。action 被 panel 重启丢弃时自动降级为纯轮询(状态行还在) |
| `preflight`         | 发布前体检(8 维度,含 `instances` drift 检查:state.hosts vs panel 实际 instance 分布),exit 0=ready / 2=有 gap / 1=挂;`gaps[].resolve` ∈ {host-pick, host-enroll, source-pick, expose-domain, cert-enable-https, dns-account-add, acme-account-add} |
| `doctor`            | 任意时刻诊断,跟 preflight 互补          |

## 组合查询(纯读)

| 命令                             | 返回                                              |
| -------------------------------- | ------------------------------------------------- |
| `site-certs check <domain>`      | `{ exact, wildcard_match, not_after }`            |
| `hosts components check <host>`  | `{ online, agent, disk, mem, components{...} }`   |
| `dns-accounts list --for <fqdn>` | 按 zone 反查匹配凭据                              |
| `probe <url>`                    | http-01 可达性                                    |

## 外部 CLI(用户本机)

| 工具 | 用途 |
| --- | --- |
| `aliyun` | `host-enroll-aliyun` 使用;查 ECS、读/写安全组、通过云助手 RunCommand 执行接管脚本。只复用用户本机 profile,不把 AccessKey 写入 panel / repo。 |
| `tccli` | `host-enroll-tencent` 使用;查 CVM、读/写安全组、通过 TAT RunCommand 执行接管脚本。只复用用户本机 profile,不把 SecretId / SecretKey 写入 panel / repo。 |

## 全局约定

- **EE-only**:`/api/v1/cli` 仅 EE panel 提供;CE panel 返回 reason `panel.no_cliapi`,此时告知用户换 EE 或走 UI,不要重试
- **`[app]` 位置参数**:app 作用域命令(`preflight` / `doctor` / `status` / `logs` / `push` / `env list` / `app-versions latest|list` / `ci-events list` / `app-instances list`)都接受可选位置参数,与 `--app` 等价且优先;缺省回落 `state.yaml::app`。组合查询(`site-certs check <domain>` / `probe <url>`)同为位置参数
- `--output json` 全局支持
- exit code:`0=done / 2=needs_input(gaps 在 JSON 里) / 1=error`
- panel URL / token 默认读 `~/.fleet/credentials`,不在每条命令重传

## 自检 · 不确定时优先查 help

本表写错或滞后(CLI 实际命令为准),AI 在选 flag / 子命令前**记不清就先 self-check**:

- `fleet help` —— 顶层所有命令族
- `fleet <族> --help` —— 列子命令(如 `fleet auth --help` 看 login / logout / set-password / whoami)
- `fleet <族> <子命令> --help` —— 该叶子命令所有 flag + 默认值
- `fleet help <cmd>` 与 `fleet <cmd> --help` 等价

不要靠记忆猜 flag、不要靠解析报错文本反推用法 —— 先 `--help`,再下命令。
