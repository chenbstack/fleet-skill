---
id: host-enroll
title: 加新机器(panel 远程 enroll)
---

# host-enroll

panel enrollment 当前不走纯 REST(stateful WebSocket 流程)。AI 不能替操作。

## Step 1 · 探测

```sh
fleet hosts list --output json
```

如已有 online host 满足需求(operator 之前手工录过)→ 直接返回,workflow 结束。

## Step 2 · 引导 operator 上 UI

CLI 调 `fleet hosts add` 会返:

```json
{
  "code": "UNSUPPORTED",
  "reason": "host.enroll_via_ui",
  "message": "host enrollment over CLI not yet wired; add host via panel UI then re-run"
}
```

AI inline 提示用户:

```
当前 host enrollment 必须在 panel UI 上做(SSH 凭据 + 实时进度需要交互)。
请打开 panel → 主机 → 录入,完成后回我一句 "ok",我再 re-run。
```

等用户回 `ok` / `done` 后,**回到调用方 workflow**(通常是 `app-publish`)
的 host 解析 step。

## Step 3 · 组件检测

录入后:

```sh
fleet hosts components check <host> --output json
```

返 `{ docker, openresty, registry_pull, buildx }` 四个 bool。处理分两路:

**3a · docker 缺**(`component.docker_missing` 隐含场景) —— 直接装,流式可见:

```sh
fleet hosts components install <host> --component=docker --output json
# → { action_id: "...", total_steps: N }
fleet exec-actions watch <action_id>
```

ExecAction 跑完 `fleet hosts components check` 再确认一次。

**3b · openresty / buildx / registry_pull 缺** —— CLI 拒装,返对应 reason:

| reason | 处理 |
| --- | --- |
| `component.openresty_missing` | UI → Sites → "Install OpenResty" 一键流(template 装 app instance,非裸 shell)|
| `component.buildx_missing` | 通常跟 docker engine 一起;重装 docker 即可,或 UI 主机详情触发 |
| `component.registry_unreachable` | UI → Image Registries 添加凭据 → `fleet hosts components check` 再验 |

inline 推回 operator,等回 `ok` 后 re-check。

## FAQ

**Q:panel UI 上录完了,CLI `fleet hosts list` 还是看不到新 host?**
A:agent 注册有 1-3s 延迟。第一次能看到时,`status` 可能还是 `pending`;再 5s 后
会变 `online`。如果 30s 后还 `pending`,通常是 agent 二进制和 panel 版本不兼容
(`/healthz` 看 panel 版本,agent 自升级里的版本对一下)。

**Q:`components check` 返 `docker: false`,但我 SSH 上去 `docker --version` 是好的?**
A:agent 跑的 `docker version` 用的是 agent 进程的 PATH。常见原因:
docker 装在 `/usr/local/bin` 但 systemd unit 没 inherit。让 operator 检查
`systemctl show fleet-agent | grep Path`。

**Q:能不能跳过 panel UI 录入,自己 SSH 上去手装 agent?**
A:可以。CE 仓 `agents/` 有预编译二进制 + `install.sh`,直接拷上去跑就行。
但 panel 仍然需要看到 host —— agent 启动后会主动 register 到 `panel.grpc_addr`,
不需要 panel 这边录入。前提:agent register 时携带的 token 是 panel 接受的。

**Q:`PostHosts` 返 `host.enroll_via_ui` 让我走 UI,什么时候会改成纯 REST?**
A:enrollment 是 stateful WebSocket 流程(SSH 凭据交互 + 实时进度 + 凭据丢弃),
塞进一次 POST 不安全也不友好。短期不会改;workaround 是上面这条手装 agent。
