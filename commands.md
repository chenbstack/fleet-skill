# CLI 命令速查

完整定义见 [`fleetpanel/docs/vibe-deploy.md §3`](https://github.com/fleetpanel/fleetpanel/blob/main/docs/vibe-deploy.md)。本表只列名字 + 一句话用途,供 AI 选命令时快速对应。

## 原子(一条 = 一个 panel REST)

| 命令族              | 子命令                                   |
| ------------------- | ---------------------------------------- |
| `auth`              | `login` / `logout` / `set-password`     |
| `tokens`            | `list` / `create` / `revoke <id>`       |
| `hosts`             | `list` / `add` ⚠️ / `get <id>` / `remove <id>` —— `add` 暂走 panel UI(后端返 `host.enroll_via_ui`),CLI 入口保留但实际入网由 UI 完成 |
| `hosts components`  | `check <host>` / `install <host>`(docker · openresty · registry_pull · buildx 四件套) |
| `dns-accounts`      | `list` / `add` / `remove <id>`          |
| `acme-accounts`     | `list` / `add` / `remove <id>`          |
| `site-certs`        | `list` / `check <domain>` / `issue` / `renew <id>` / `remove <id>` —— 无独立 `install`,`issue` 已含落地到 OpenResty |
| `image-registries`  | `list` / `add` / `remove <id>`          |
| `app-instances`     | `list` / `create` / `get <id>` / `remove <id>` / `scale <id>` / `rollback <id>` —— 无 `update`,改 hosts/replicas 走 `scale` |
| `app-builds`        | `list` / `get <id>`                     |
| `app-versions`      | `latest` / `list`                       |
| `ci-events`         | `list` / `get <id>`                     |
| `exec-actions`      | `watch <id>` / `list`                   |
| `logs`              | `--build <id>` / `--tail` / `--level error` |
| `status`            | `[--metrics] [--wait <duration>]`       |
| `env`               | `list` / `set NAME=VALUE` / `remove NAME` |
| `init`              | `--app <name>` / `--host <label>`(可重复)/ `--source tarball\|dockerfile\|image\|static` / `--static-dir <dir>` / `--build panel\|ci\|external` / `--version-policy semver\|external\|calver` / `--domain` / `--tls` / `--panel <url>` / `--force` |
| `push`              | `--version vX.Y.Z`(必填,见 D20);`state.source=static` 时 `--version` 不强制,自动 tar.gz `state.static_dir`(默认 `./dist`)上传到 `/apps/{app}/site:upload` |
| `preflight`         | 发布前体检,exit 0=ready / 2=有 gap / 1=挂 |
| `doctor`            | 任意时刻诊断,跟 preflight 互补          |

## 组合查询(纯读)

| 命令                             | 返回                                              |
| -------------------------------- | ------------------------------------------------- |
| `site-certs check <domain>`      | `{ exact, wildcard_match, not_after }`            |
| `hosts components check <host>`  | `{ online, agent, disk, mem, components{...} }`   |
| `dns-accounts list --for <fqdn>` | 按 zone 反查匹配凭据                              |
| `probe <url>`                    | http-01 可达性                                    |

## 全局约定

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
