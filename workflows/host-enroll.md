---
id: host-enroll
title: 加新机器(云厂商优先,token 脚本 fallback)
sub_workflows: [host-buy-aliyun, host-enroll-aliyun, host-enroll-tencent]
---

# host-enroll

先问云厂商,不要一上来要求用户复制 IP / 用户名 / 密码。用户已经说了云厂商
时直接跳到对应分支。

## Step 1 · 探测

```sh
fleet hosts list --output json
```

如已有 online host 满足需求(operator 之前手工录过)→ 直接返回,workflow 结束。

## Step 2 · 选择云厂商入口

inline 让用户选(先问有没有服务器,没有就帮买):

```
[1] 我还没有服务器 —— 帮你在阿里云买一台(对话里选配置、扫码支付、自动接入)
[2] 阿里云 ECS —— 我用你本机 aliyun CLI 查实例 / 开安全组 / 走云助手安装
[3] 腾讯云 CVM —— 我用你本机 tccli 查实例 / 开安全组 / 走自动化助手安装
[4] 其他 / 自建 / SSH 地址 —— 我生成一段安装脚本,在机器上以 root 跑一次即可
```

- 选 `1` → 套 [`host-buy-aliyun`](host-buy-aliyun.md),完成后回到 Step 4。
- 选 `2` → 套 [`host-enroll-aliyun`](host-enroll-aliyun.md),完成后回到 Step 4。
- 选 `3` → 套 [`host-enroll-tencent`](host-enroll-tencent.md),完成后回到 Step 4。
- 选 `4` → 继续 Step 3。

## Step 3 · token 脚本 fallback(任意能跑 sh 的 linux)

```sh
fleet hosts enroll-script --name <机器名> --output json
# → { host_id, script, expires_at }   token 一次性,24h 有效
```

把 `script` 存成文件交给用户(或经用户授权的 ssh 直接执行),在目标机以
root 跑一次:下载 agent → 写配置 → 装 systemd/openrc 服务 → 自动注册。
脚本幂等,失败可重跑;**内容含一次性 token,不要贴进最终回复或日志**。
panel 用 https 时证书必须公网受信(Let's Encrypt 即可),脚本会正常校验
证书,自签会下载失败;内网/dev 可显式给 `--panel-url http://…`(明文传
token,风险自担)。

```
我生成了一段安装脚本(含一次性接入凭证,24 小时有效)。
在你的服务器上以 root 执行它即可,完成后回我一句 "ok",我来确认上线。
```

之后 `fleet hosts list --output json` 轮询 `host_id` 变 online → Step 4。
panel UI(主机 → 录入,SSH 凭据 + 实时进度)仍可用,适合不方便跑脚本的用户。

## Step 4 · 组件检测

录入后:

```sh
fleet hosts components check <host> --output json
```

返 `{ docker, openresty, registry_pull, buildx }` 四个 bool。处理分两路:

**4a · docker 缺**(`component.docker_missing` 隐含场景) —— 直接装,流式可见:

```sh
fleet hosts components install <host> --components docker --output json
# → { action_id: "...", total_steps: N }
fleet exec-actions watch <action_id>
```

ExecAction 跑完 `fleet hosts components check` 再确认一次。

**4b · openresty / buildx / registry_pull 缺** —— CLI 拒装,返对应 reason:

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
A:可以,而且不用手装 —— `fleet hosts enroll-script` 生成的脚本就是为这个
场景准备的(token 注册 + 下载 + 服务安装一条龙)。Step 3 即此路径。

**Q:`fleet hosts add` 返 `host.enroll_via_ui` 是什么意思?**
A:`hosts add`(panel 代登 SSH 的 enrollment)是 stateful WebSocket 流程,
没有 REST 形态。非交互场景一律用 `fleet hosts enroll-script`;需要 panel
代登 SSH(用户只有 IP + 密码、且不方便自己跑脚本)时走 panel UI。

**Q:enroll-script 生成的 pending host 一直没上线,会留垃圾吗?**
A:token 24h 过期,pending 行不会自己消失;确认不再用时
`fleet hosts remove <host_id>` 清掉,重新生成即可。
