---
id: host-enroll-tencent
title: 腾讯云 CVM 本地 CLI 接管
---

# host-enroll-tencent

在用户本机调用 `tccli`。不要把腾讯云 SecretId / SecretKey 写入 panel、仓库或
任何项目配置(`~/.fleet` 注册表 / `.fleet/state.yaml`);只复用用户本机已有
profile。写操作前必须列出将要改的
实例 / 安全组 / 端口,等用户确认。

## Step 0 · CLI 与权限预检

```sh
command -v tccli
tccli configure list
```

- `tccli` 不存在 → 让用户先安装 / 配置 TCCLI。
- region 缺失 → 让用户补 `tccli configure`,或 inline 询问 region
  (如 `ap-guangzhou`)。
- CAM 至少需要:
  `cvm:DescribeInstances`, `vpc:DescribeSecurityGroups`,
  `vpc:DescribeSecurityGroupPolicies`, `vpc:CreateSecurityGroupPolicies`,
  `tat:RunCommand`, `tat:DescribeInvocationTasks`。

不确定参数时先查:

```sh
tccli cvm DescribeInstances help
tccli vpc CreateSecurityGroupPolicies help
tccli tat RunCommand help
```

## Step 1 · 列 CVM 让用户选

```sh
tccli cvm DescribeInstances \
  --region <region> \
  --Limit 100 \
  --Filters '[{"Name":"instance-state","Values":["RUNNING"]}]' \
  --output json
```

从 JSON 的 `Response.InstanceSet[]` 里列:

- `InstanceId`
- `InstanceName`
- 公网 IP:`PublicIpAddresses[0]`
- 私网 IP:`PrivateIpAddresses[0]`
- `SecurityGroupIds[]`
- `OsName`
- `DefaultLoginUser` / `DefaultLoginPort`
- `InstanceState`

让用户选实例。不要替用户在多台机器里猜。没有公网 IP 时说明:
「这台 CVM 没有公网 IP;如果 panel 与 CVM 不在同一内网,需要先绑定公网 IP 或
走控制台 / SSH fallback。」

## Step 2 · 安全组规则

先读取现有规则,避免重复或放大暴露面:

```sh
tccli vpc DescribeSecurityGroupPolicies \
  --region <region> \
  --SecurityGroupId <sg-id> \
  --output json
```

默认只建议最小规则:

- `22/tcp` 来源为 operator 当前公网 IP `/32`,仅当后续需要 SSH fallback。
- `80/tcp` 和 `443/tcp` 来源按用户意图;公网 Web 服务才用 `0.0.0.0/0`。
- 不默认开放 panel 管理端口、数据库端口、Docker API 或全端口。

写入前 inline 确认:

```
将修改腾讯云安全组 <sg-id>:
- TCP 22 from <your-ip>/32,用于接管 fallback
- TCP 80,443 from 0.0.0.0/0,用于公网 Web
确认执行? [1] 执行 [2] 跳过
```

确认后执行。腾讯云安全组规则端口格式是 `22` 或 `80-443`,不是阿里云的
`22/22`:

```sh
tccli vpc CreateSecurityGroupPolicies \
  --region <region> \
  --SecurityGroupId <sg-id> \
  --SecurityGroupPolicySet '{"Ingress":[{"Protocol":"TCP","Port":"22","CidrBlock":"<operator-ip>/32","Action":"ACCEPT","PolicyDescription":"fleet-enroll-ssh"}]}' \
  --output json
```

`80`、`443` 同理。若返回 `UnsupportedOperation.DuplicatePolicy`,视为已存在。

## Step 3 · 优先走自动化助手 TAT

目标是绕过 SSH 密码。腾讯云 API 不会返回已有 CVM 的明文密码,所以不要向用户
承诺“自动拿密码”。优先用 TAT `RunCommand` 执行接管脚本。

先验证 TAT 可用:

```sh
tccli tat RunCommand \
  --region <region> \
  --Content "$(printf '%s' 'echo fleet-tat-ok' | base64)" \
  --InstanceIds '["<instance-id>"]' \
  --CommandType SHELL \
  --SaveCommand false \
  --Timeout 60 \
  --output json
```

如果返回未安装 agent、agent 不在线、实例非 VPC 或非 RUNNING,提示用户处理 TAT
Agent 或回到 [`host-enroll`](host-enroll.md) 的 UI / SSH fallback。

接管脚本从 panel 拿(一次性 token,24h 有效;脚本幂等可重跑):

```sh
fleet hosts enroll-script --name <实例名> \
  [--panel-addr <公网host:7443>] [--panel-url https://<公网host>] \
  --output json
# → { host_id, script, expires_at }
```

注意 CVM 是从公网连 panel:`panel_addr`(agent gRPC 回连)和 `panel_url`
(下载 agent 二进制)都必须是 CVM 可达的地址,panel 在内网/本机时要用 flag
覆盖。不要把 token / 脚本内容打印到最终回复;脚本必须 Base64 后传 `Content`:

```sh
tccli tat RunCommand \
  --region <region> \
  --Content <base64-script> \
  --InstanceIds '["<instance-id>"]' \
  --CommandType SHELL \
  --Username root \
  --SaveCommand false \
  --Timeout 1800 \
  --output json
```

RunCommand 成功会返回 `InvocationId`。查询任务结果:

```sh
tccli tat DescribeInvocationTasks \
  --region <region> \
  --Filters '[{"Name":"invocation-id","Values":["<invocation-id>"]}]' \
  --HideOutput false \
  --output json
```

若结果还在 running,等 3-5s 后再查;不要无限循环,同一命令连续 10 次未终态就
停下来把 `InvocationId` 给用户。

## Step 4 · 回 panel 验证 host

TAT 脚本成功后:

```sh
fleet hosts list --output json
```

用 CVM 公网 IP / 私网 IP / hostname / agent 上报的 cloud=`tencent`、region
匹配新 host。匹配到 online 或 pending host 后回到
[`host-enroll`](host-enroll.md) Step 4 做组件检测。

30s 后仍无 host:

- 检查安全组出站是否放行到 panel。
- 检查 panel URL / gRPC 地址是否从 CVM 可达。
- 用 `DescribeInvocationTasks` 看脚本 stdout / stderr。
- 仍失败则回 UI / SSH fallback。

## 禁区

- 不调用重置实例密码、停止、重启、销毁实例等破坏性 API,除非用户明确要求。
- 不把 SSH `CidrBlock` 默认设成 `0.0.0.0/0`。
- 不保存 SecretId / SecretKey;不把 TAT 脚本里的 token 写入仓库、日志或任何
  项目配置(`~/.fleet` 注册表 / `.fleet/state.yaml`)。
