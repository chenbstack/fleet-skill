---
id: host-enroll-aliyun
title: 阿里云 ECS 本地 CLI 接管
---

# host-enroll-aliyun

在用户本机调用 `aliyun` CLI。不要把阿里云 AccessKey 写入 panel、仓库或
任何项目配置(`~/.fleet` 注册表 / `.fleet/state.yaml`);只复用用户本机已有
profile。写操作前必须列出将要改的
实例 / 安全组 / 端口,等用户确认。

## Step 0 · CLI 与权限预检

```sh
command -v aliyun
aliyun configure list
```

- `aliyun` 不存在 → 让用户先安装 / 配置阿里云 CLI,不要改走 SDK。
- profile / region 缺失 → 让用户补 `aliyun configure`,或 inline 询问 region
  (如 `cn-hangzhou`)。
- RAM 至少需要:
  `ecs:DescribeInstances`, `ecs:DescribeSecurityGroupAttribute`,
  `ecs:AuthorizeSecurityGroup`, `ecs:RunCommand`,
  `ecs:DescribeInvocationResults`。

不确定参数时先查:

```sh
aliyun ecs DescribeInstances --help
aliyun ecs AuthorizeSecurityGroup --help
aliyun ecs RunCommand --help
```

## Step 1 · 列 ECS 让用户选

```sh
aliyun ecs DescribeInstances \
  --RegionId <region> \
  --Status Running \
  --PageSize 100 \
  --output json
```

从 JSON 里列:

- `InstanceId`
- `InstanceName`
- 公网 IP:优先 EIP,其次 `PublicIpAddress.IpAddress[0]`
- 私网 IP:`VpcAttributes.PrivateIpAddress.IpAddress[0]`
- `SecurityGroupIds.SecurityGroupId[]`
- `OSType` / `OSName` / `Status`

让用户选实例。不要替用户在多台机器里猜。没有公网 IP 时说明:
「这台 ECS 没有公网 IP;如果 panel 与 ECS 不在同一内网,需要先绑定 EIP 或走
控制台 / SSH fallback。」

## Step 2 · 安全组规则

先读取现有规则,避免重复或放大暴露面:

```sh
aliyun ecs DescribeSecurityGroupAttribute \
  --RegionId <region> \
  --SecurityGroupId <sg-id> \
  --output json
```

默认只建议最小规则:

- `22/tcp` 来源为 operator 当前公网 IP `/32`,仅当后续需要 SSH fallback。
- `80/tcp` 和 `443/tcp` 来源按用户意图;公网 Web 服务才用 `0.0.0.0/0`。
- 不默认开放 panel 管理端口、数据库端口、Docker API 或全端口。

写入前 inline 确认:

```
将修改阿里云安全组 <sg-id>:
- 22/tcp from <your-ip>/32,用于接管 fallback
- 80/tcp,443/tcp from 0.0.0.0/0,用于公网 Web
确认执行? [1] 执行 [2] 跳过
```

确认后执行(每条规则单独跑,便于定位失败):

```sh
aliyun ecs AuthorizeSecurityGroup \
  --RegionId <region> \
  --SecurityGroupId <sg-id> \
  --IpProtocol tcp \
  --PortRange 22/22 \
  --SourceCidrIp <operator-ip>/32 \
  --Description fleet-enroll-ssh \
  --output json
```

`80/80`、`443/443` 同理。已有规则时阿里云会幂等成功;仍要告知用户没有新增。

## Step 3 · 优先走云助手安装

目标是绕过 SSH 密码。阿里云不会返回已有 ECS 的明文密码,所以不要向用户承诺
“自动拿密码”。优先用 ECS 云助手 `RunCommand` 执行接管脚本。

先 dry-run 云助手:

```sh
aliyun ecs RunCommand \
  --RegionId <region> \
  --InstanceId.1 <instance-id> \
  --Type RunShellScript \
  --CommandContent "echo fleet-cloud-assistant-ok" \
  --RepeatMode DryRun \
  --Timeout 60 \
  --output json
```

如果云助手不可用,提示用户安装 Cloud Assistant Agent 或回到
[`host-enroll`](host-enroll.md) 的 UI / SSH fallback。

接管脚本从 panel 拿(一次性 token,24h 有效;脚本幂等可重跑):

```sh
fleet hosts enroll-script --name <实例名> \
  [--panel-addr <公网host:7443>] [--panel-url https://<公网host>] \
  --output json
# → { host_id, script, expires_at }
```

注意 ECS 是从公网连 panel:`panel_addr`(agent gRPC 回连)和 `panel_url`
(下载 agent 二进制)都必须是 ECS 可达的地址,panel 在内网/本机时要用 flag
覆盖。不要把 token / 脚本内容打印到最终回复;命令内容 Base64 后设置
`ContentEncoding=Base64`:

```sh
aliyun ecs RunCommand \
  --RegionId <region> \
  --InstanceId.1 <instance-id> \
  --Type RunShellScript \
  --CommandContent <base64-script> \
  --ContentEncoding Base64 \
  --RepeatMode Once \
  --Timeout 1800 \
  --output json
```

RunCommand 是异步接口。拿返回的 `InvokeId` / `CommandId` 后查结果:

```sh
aliyun ecs DescribeInvocationResults \
  --RegionId <region> \
  --InvokeId <invoke-id> \
  --output json
```

若结果还在 running,等 3-5s 后再查;不要无限循环,同一命令连续 10 次未终态就
停下来把 `InvokeId` 给用户。

## Step 4 · 回 panel 验证 host

云助手脚本成功后:

```sh
fleet hosts list --output json
```

用 ECS 公网 IP / 私网 IP / hostname / agent 上报的 cloud=`aliyun`、region
匹配新 host。匹配到 online 或 pending host 后回到
[`host-enroll`](host-enroll.md) Step 4 做组件检测。

30s 后仍无 host:

- 检查安全组出站是否放行到 panel。
- 检查 panel URL / gRPC 地址是否从 ECS 可达。
- 用 `DescribeInvocationResults` 看脚本 stdout / stderr。
- 仍失败则回 UI / SSH fallback。

## 禁区

- 不调用重置实例密码、停止、重启、释放实例等破坏性 API,除非用户明确要求。
- 不把 `SourceCidrIp` 默认设成 `0.0.0.0/0` 开 SSH。
- 不保存 AccessKey;不把云助手脚本里的 token 写入仓库、日志或任何项目配置(`~/.fleet` 注册表 / `.fleet/state.yaml`)。
