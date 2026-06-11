---
id: host-buy-aliyun
title: 没有服务器:对话内购买阿里云 ECS 并自动接管
when_to_use: |
  用户没有任何可用服务器、确认想买一台时(host-enroll Step 2 的「买一台」
  分支)。全程在用户本机 aliyun CLI 完成:选配 → 询价 → AutoPay=false 待支付
  订单 → 用户扫码支付 → cloud-init UserData 自动装 agent 入列。
referenced_workflows:
  - host-enroll
  - host-enroll-aliyun
---

# host-buy-aliyun

四个打断点(凭据 / 选配 / 下单确认 / 扫码支付)都是只有用户能做的事,其余
全自动。不要把阿里云 AccessKey 写入 panel、仓库或 `.fleet/state.yaml`;
enroll 脚本含一次性 token,不要打印到最终回复或日志。

## Step 0 · 凭据预检(打断点 1)

```sh
command -v aliyun
aliyun configure list
```

- CLI 缺 → 让用户装(macOS `brew install aliyun-cli`),不要改走 SDK。
- 无有效 AccessKey → inline 引导(主账号不能用 OAuth,只能 AccessKey):

```
需要一次阿里云凭据配置(只存你本机 ~/.aliyun,不上传):
1. 打开 https://ram.console.aliyun.com/manage/ak 创建 AccessKey
2. 回来跑:aliyun configure(粘贴 Key/Secret,region 建议 cn-hangzhou)
完成后回我一句 "ok"。
```

- RAM 至少需要:`ecs:DescribeAvailableResource`, `ecs:DescribePrice`,
  `ecs:RunInstances`, `ecs:DescribeInstances`, `vpc:*Describe*`,
  `ecs:CreateSecurityGroup`, `ecs:AuthorizeSecurityGroup`。

## Step 1 · 选配与询价(打断点 2)

部署一两个 side project,默认推荐入门档,不要替用户拍板:

```sh
# 查该 region 哪些规格有货(IoOptimized=optimized,按量先验货架)
aliyun ecs DescribeAvailableResource --RegionId <region> \
  --DestinationResource InstanceType --InstanceChargeType PrePaid --output json
# 对候选规格逐个询价(1 个月包月)
aliyun ecs DescribePrice --RegionId <region> --ResourceType instance \
  --InstanceType ecs.e-c1m1.large --Period 1 --PriceUnit Month \
  --ImageId <image-id> --SystemDisk.Category cloud_essd --output json
```

镜像固定选 Ubuntu LTS(`aliyun ecs DescribeImages --RegionId <region>
--OSType linux --ImageOwnerAlias system --output json` 里挑
`ubuntu_24_04` 最新),用户没有理由关心这个,不要问。

inline 给 2-3 档(规格 / 月价 / 适合什么),例:

```
[1] 2核2G ecs.e-c1m1.large + 40G 盘 + 3M 带宽 ≈ ¥xx/月 —— 个人项目够用(推荐)
[2] 2核4G ecs.e-c1m2.large …… ≈ ¥xx/月 —— 跑数据库 / 多个应用
带宽默认按固定 3Mbps;价格以下一步订单页为准。
```

## Step 2 · 网络与安全组

```sh
aliyun vpc DescribeVpcs --RegionId <region> --output json
aliyun vpc DescribeVSwitches --RegionId <region> --VpcId <vpc-id> --output json
```

- 有默认 VPC/vswitch → 直接用;没有 → `CreateDefaultVpc` 一条命令补。
- 新建专用安全组,最小放行:`80/tcp`、`443/tcp` from `0.0.0.0/0`。
  **不默认开 22** —— UserData 自动接管不需要 SSH;用户明确要 SSH 时按
  [`host-enroll-aliyun`](host-enroll-aliyun.md) Step 2 的 `/32` 规则加。

```sh
aliyun ecs CreateSecurityGroup --RegionId <region> --VpcId <vpc-id> \
  --SecurityGroupName fleet-web --output json
aliyun ecs AuthorizeSecurityGroup --RegionId <region> --SecurityGroupId <sg-id> \
  --IpProtocol tcp --PortRange 80/80 --SourceCidrIp 0.0.0.0/0 --output json
# 443 同理
```

## Step 3 · 接管脚本 + 下单(打断点 3)

先从 panel 拿一次性接管脚本(panel 必须从公网可达,确认 `panel_addr` /
`panel_url` 是 ECS 能连到的地址,必要时用 flag 覆盖;https 证书必须公网
受信,脚本会正常校验,自签会下载失败):

```sh
fleet hosts enroll-script --name <用户起的机器名> \
  [--panel-addr <公网host:7443>] [--panel-url https://<公网host>] \
  --output json
# → { host_id, script, expires_at }   token 有效期 24h,一次性
```

`script` 整段 base64 后作为 UserData(cloud-init 首次开机 root 执行):

```sh
base64 -i <script文件>   # macOS;linux 用 base64 -w0
```

下单前 inline 复述全部要花钱的项并确认:

```
即将创建包月订单(AutoPay=false,只生成待支付订单,不会直接扣款):
- 规格 … / 镜像 Ubuntu 24.04 / 盘 40G ESSD / 带宽 3M
- 地域 cn-hangzhou / 安全组 fleet-web(80,443)
- 月价 ≈ ¥xx,支付后自动开机并自动接入面板
确认下单? [1] 下单 [2] 改配置 [3] 放弃
```

```sh
aliyun ecs RunInstances --RegionId <region> \
  --InstanceType <type> --ImageId <image-id> \
  --SecurityGroupId <sg-id> --VSwitchId <vsw-id> \
  --InstanceName <name> --HostName <name> \
  --InternetMaxBandwidthOut 3 --InternetChargeType PayByBandwidth \
  --SystemDisk.Size 40 --SystemDisk.Category cloud_essd \
  --InstanceChargeType PrePaid --Period 1 --PeriodUnit Month \
  --AutoPay false \
  --UserData <base64-script> \
  --output json
# → { InstanceIdSets: [...], OrderId: ... }
```

## Step 4 · 扫码支付(打断点 4)

```
订单已创建(未支付,订单号 <OrderId>)。
打开 https://usercenter2.aliyun.com/order/list 或阿里云 App 扫码完成支付;
支付后实例会自动开机并在 1-3 分钟内出现在面板里。我每 30s 检查一次。
```

轮询(30s 一次,最多 20 次;超时把订单号留给用户,不要 `CancelOrder` ——
取消订单只在用户明说反悔时做):

```sh
aliyun ecs DescribeInstances --RegionId <region> \
  --InstanceIds '["<instance-id>"]' --output json   # Status: Running 即已付款开机
```

## Step 5 · 等 agent 自动入列

实例 Running 后,cloud-init 会跑 UserData 里的接管脚本(下载 agent →
注册)。轮询 panel:

```sh
fleet hosts list --output json   # 看 Step 3 拿到的 host_id 是否 online
```

online → 回 [`host-enroll`](host-enroll.md) Step 4 做组件检测,workflow 结束。

## Step 6 · 兜底:UserData 没生效

实例 Running 超过 5 分钟 host 仍 pending:用云助手重跑同一段脚本(脚本幂等,
重复执行安全;token 未被消费就仍有效):

```sh
aliyun ecs RunCommand --RegionId <region> --InstanceId.1 <instance-id> \
  --Type RunShellScript --CommandContent <base64-script> \
  --ContentEncoding Base64 --RepeatMode Once --Timeout 600 --output json
aliyun ecs DescribeInvocationResults --RegionId <region> --InvokeId <invoke-id> --output json
```

仍失败 → 用 `DescribeInvocationResults` 的 stdout/stderr 定位(常见:panel
地址从公网不可达 / token 过期 24h);token 过期就重新 `fleet hosts
enroll-script` 生成,旧 pending 行用 `fleet hosts remove` 清掉。

## 禁区

- 不保存 / 转述 AccessKey;它只存在用户本机 `~/.aliyun`。
- enroll 脚本含一次性 token:不打印进最终回复,不写仓库 / 日志 / state.yaml。
- 不默认开 22 端口;要开必须 `/32` 来源 + 用户确认。
- `CancelOrder` / 退款 / 释放实例等只在用户明确要求时执行。
- 同一会话里不重复下单:RunInstances 失败重试前先 `DescribeInstances` +
  订单列表确认没有半成品。
