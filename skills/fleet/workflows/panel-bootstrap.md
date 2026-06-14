---
id: panel-bootstrap
title: 冷启动:还没有 fleet 面板(买机装面板 / 已有服务器装面板 / 已有面板登录)
when_to_use: |
  fleet CLI 在但 `~/.fleet/credentials` 不存在(或 `fleet auth whoami` 连不上
  任何 panel),且用户的目标需要一个面板才能继续(部署 / 加机器 / 一切)。
  这是所有 workflow 的最上游 —— 没面板时先走这里,装好登上再回原 workflow。
referenced_workflows:
  - host-buy-aliyun
  - host-enroll
  - app-publish
---

# panel-bootstrap

面板装在用户自己的服务器上(没有就买一台),panel 与第一台应用宿主机同机
(`--standalone`):面板既是控制面也直接跑应用,以后加机器再横向扩。
没有 license 时面板以 **14 天全功能试用** 运行;到期后已接入的主机照常管理,
只是不能再添加新主机。

凭据纪律(全程):装面板前先做一次「参数预确认」(见 Step 2A)让用户自选管理
端口 / 入口路径 / 用户名 / 密码,用户没指定的项由你(AI)在用户本机随机生成;
所有值经命令参数(env)注入安装脚本,install.sh 对注入过的项不会再交互追问
—— **不写仓库 / 项目配置 / 日志,中间步骤不打印**;但安装完成后**必须**在
最终回复里把面板地址和完整凭据一次性交给用户(这是用户唯一的获取渠道),用了
随机密码时建议尽快改密。

## Step 0 · 探测

```sh
test -f ~/.fleet/credentials && fleet auth whoami --output json
```

whoami 正常返回 → 已有面板,不该进本 workflow,回到原任务。
credentials 不存在 / panel 连不上 → Step 1。

## Step 1 · 入口三选一(打断点)

```
还没有连接任何 fleet 面板。你的情况是?
[1] 我一台服务器都没有 —— 帮你买一台阿里云 ECS,把面板装在上面(开机即用,推荐)
[2] 我有服务器 —— 在它上面装面板(你跑一条命令,或授权我 SSH 执行)
[3] 已经有 fleet 面板了 —— 直接登录
```

- 选 `1` → Step 2A;选 `2` → Step 2B;选 `3` → Step 2C。

## Step 2A · 买机 + 装面板

复用 [`host-buy-aliyun`](host-buy-aliyun.md) 的 Step 0(凭据预检)、Step 1
(选配询价)、Step 2(网络与安全组),差异只有两处:

1. **安全组**额外放行两个端口(都来自下面生成的配置):
   - `<管理端口>/tcp` from `0.0.0.0/0` —— 面板 HTTP(入口路径是 12 位随机
     hex,扫描器拿不到 path 只会看到 fake 404)
   - `7443/tcp` from `0.0.0.0/0` —— 面板 gRPC,以后添加远程主机时 agent 回连
     (默认 7443;若 UserData 注入了 `FLEETPANEL_GRPC_PORT` 则改放行那个端口)
2. **UserData 不是 enroll 脚本**,而是面板引导脚本(见下)。

**参数预确认**(打断点,四项一起问;用户回「都随机」/ 不在乎 = 全部由你生成):

```
面板入口的 4 个参数,想自定义哪些?不指定的项我会用安全随机值:
- 管理端口(1024-65535;别撞 80/443/7443)
- 入口路径(面板 URL 的隐藏前缀,建议 12 位字母数字)
- 管理员用户名(默认 admin)
- 管理员密码(不说就生成 16 位随机密码)
```

用户给了的直接采用;没给的本地生成并记住(后面 login / 交付用户都要):

```sh
PORT=$(( (RANDOM % 55000) + 10000 ))   # 避开 80/443/7443;zsh/bash 均可
ENTRANCE=$(head -c 6 /dev/urandom | xxd -p)        # 12 位 hex
ADMIN_USER=admin
ADMIN_PW=$(LC_ALL=C tr -dc 'A-Za-z0-9!@%*_-' </dev/urandom | head -c 16)
```

UserData 模板(cloud-init 首次开机 root 执行;阿里云公网 IP 是 NAT 映射,
不在网卡上,要从 metadata 服务拿):

```sh
#!/bin/sh
export FLEETPANEL_HTTP_PORT='<PORT>'
export FLEETPANEL_ENTRANCE='<ENTRANCE>'
export FLEETPANEL_ADMIN_USER='<ADMIN_USER>'
export FLEETPANEL_ADMIN_PW='<ADMIN_PW>'
# agent 网关端口默认 7443(全新机一定空闲,通常不用设)。只有要避开 7443 时
# 才取消注释填一个空闲端口,并记得安全组同步放行它(见上一条):
# export FLEETPANEL_GRPC_PORT='<GRPC_PORT>'
PUB=$(curl -s --max-time 5 http://100.100.100.200/latest/meta-data/eipv4 || true)
[ -n "$PUB" ] || PUB=$(curl -s --max-time 5 http://100.100.100.200/latest/meta-data/public-ipv4 || true)
[ -n "$PUB" ] && export FLEETPANEL_PUBLIC_HOST="$PUB"
curl -fsSL https://git.puzizi.cn/launchpad/fleetpanel-ee/-/raw/main/install.sh | sh
```

base64 后作 UserData,下单 / 扫码支付 / 轮询 Running 全同
[`host-buy-aliyun`](host-buy-aliyun.md) Step 3-4(同样 AutoPay=false、同样的
确认话术,把「自动接入面板」换成「自动装好面板」)。

实例 Running 后拿公网 IP(`DescribeInstances` 的
`PublicIpAddress` / `EipAddress`),轮询面板起来(装机 1-3 分钟):

```sh
curl -fsS --max-time 5 "http://<公网IP>:<PORT>/<ENTRANCE>/healthz"
# 30s 一次,最多 10 分钟;返回 JSON(含版本号)即就绪
```

超时 → 云助手 `RunCommand` 跑 `journalctl -u fleet-panel -e --no-pager` 定位
(参考 host-buy-aliyun Step 6 的 RunCommand 用法);也看 cloud-init 日志
`journalctl -u cloud-final --no-pager`,install.sh 装前预检若报「端口已被占用」
会在这里 die。命中端口占用 → 见 FAQ「装机报端口被占用」。

就绪后 → Step 3。

## Step 2B · 已有服务器装面板

先确认:linux、root 权限、对外可达 IP(或内网场景用户自己明白可达性)。
参数预确认 + 没给的项本地生成,同 Step 2A。

让用户在服务器上跑(`sudo` 默认重置环境变量,必须用 `sudo env` 透传):

```sh
curl -fsSL https://git.puzizi.cn/launchpad/fleetpanel-ee/-/raw/main/install.sh \
  | sudo env FLEETPANEL_PUBLIC_HOST='<服务器公网IP或域名>' \
      FLEETPANEL_HTTP_PORT='<PORT>' FLEETPANEL_ENTRANCE='<ENTRANCE>' \
      FLEETPANEL_ADMIN_USER='<ADMIN_USER>' FLEETPANEL_ADMIN_PW='<ADMIN_PW>' sh
```

(用户授权 SSH 时可以由你直接执行同一条;不要把这条命令写进日志 / 仓库。
install.sh 对 env 注入过的项不会再交互追问,所以这条命令粘进用户终端执行
也不会被提问打断;反之漏注入的项脚本会在终端里自己问。)
脚本幂等:已有 `/opt/fleet/panel/config.yaml` 时只升级二进制不动配置。

提醒用户云安全组 / 防火墙放行 `<PORT>/tcp` 与 `7443/tcp`(同 Step 2A 理由),
然后 healthz 轮询同 Step 2A → Step 3。若命令以「端口已被占用」报错退出
(install.sh 装前预检拦下,没装出半成品)→ 见 FAQ「装机报端口被占用」。

## Step 2C · 已有面板,直接登录

先问用户手里有什么凭据,两条路:

- **有 PAT**(`fpat_…`,通常来自面板「访问凭证」页的交付文本)→ 只需面板
  URL(**含入口路径**,形如 `http://<host>:<port>/<entrance>`)+ token:

  ```sh
  fleet auth login --panel "http://<host>:<port>/<entrance>" --with-token "fpat_…"
  fleet auth whoami --output json   # 确认
  ```

  成功即完成登录,跳过 Step 3 的 login,直接回原任务。
- **只有用户名 / 密码** → 问拿三样:面板 URL(含入口路径)、用户名、密码
  → 进 Step 3 的 login。

两条路的 URL 都必须带入口路径 —— 缺了会吃 entrance 防护的 fake 404,CLI
报 `panel.no_cliapi` 误导你以为是 CE panel(见 SKILL.md「前提:EE panel」)。

## Step 3 · 可达性验证 + 登录 + 交付

交付前先替用户验证哪个地址真能打开,**在用户本机**逐个试(这里通了,用户的
浏览器才打得开)。候选按序:公网 IP → 内网 IP(服务器上 `hostname -I` 第一
块;install.sh 收尾输出也会公网/内网各给一条):

```sh
curl -fsS --max-time 5 "http://<host>:<PORT>/<ENTRANCE>/healthz"
```

- 公网通 → 用公网地址交付(最常见)。
- 公网不通、内网通(用户与服务器同内网 / VPN)→ 用内网地址交付,并说明公网
  为何不可达(未绑公网 IP / 安全组没放行)。
- 都不通 → 回查云安全组与服务器本机防火墙(install.sh 已自动放行本机
  firewalld / ufw;云安全组要用户在控制台放行 `<PORT>/tcp` + `7443/tcp`),
  修通再交付。**不要把没验证过的地址交给用户**。

```sh
fleet auth login --panel "http://<验证通过的host>:<PORT>/<ENTRANCE>" \
  --user '<ADMIN_USER>' --password '<ADMIN_PW>'
fleet auth whoami --output json   # 确认
```

最终回复里一次性交付(仅这一次,之后不再出现在任何输出里):

```
面板已装好并登录成功(14 天全功能试用,到期后已接入的主机不受影响,
只是不能再加新主机;购买 license 后放到服务器 /opt/fleet/panel/data/fleetpanel.license)。

  面板地址: http://<验证通过的host>:<PORT>/<ENTRANCE>/
  用户名:   <ADMIN_USER>
  密码:     <ADMIN_PW>

请保存好,建议尽快在面板里修改密码(服务器上 /opt/fleet/panel/credentials.txt
有同一份,记完请删)。
```

买机 / 装机分支(2A/2B)装的是 standalone 面板:本机已经自动作为第一台主机
接入,**不需要**再走 host-enroll —— 直接回原任务(通常是
[`app-publish`](app-publish.md));要加更多机器时才走
[`host-enroll`](host-enroll.md)。

## FAQ

**Q:healthz 一直 404?**
A:404 是 entrance 防护的 fake 页 —— 说明面板活着但 path 不对,核对
`<ENTRANCE>` 是否与注入值一致(别丢了前导路径)。连接拒绝才是没起来。

**Q:装机报端口被占用 / 面板装不上?**
A:install.sh 装前会预检管理端口和 7443,任一被目标机已有进程占用即 `die`
中止 —— 不会装出一个起不来的面板,所以这是「拦在装之前」,不是坏状态。处理:
- **管理端口撞了** → 换一个没人用的端口(目标机上 `ss -ltn` 看占用,或本地
  重新随机一个),把 `<PORT>` 换掉重跑同一条命令:2A 改 UserData 里的
  `FLEETPANEL_HTTP_PORT` 后重装(机内重跑 install.sh,或重建实例),2B 直接
  用新端口重跑那条 `curl | sudo env … sh`。**新端口记得同步在云安全组 /
  防火墙放行**,旧端口可不管。
- **7443 撞了** → agent 网关端口现在可配。install.sh 默认用 7443,被占时
  交互安装会提示你换;无人值守(2A UserData / 2B 非交互)就注入
  `FLEETPANEL_GRPC_PORT=<空闲端口>` 重装(`ss -ltnp` 看占用)。换了网关端口
  记得云安全组放行新端口(本机防火墙 install.sh 会按新端口自动放行)。
  腾出 7443 仍是另一条路,但不再是唯一出路。

**Q:试用过期了会怎样?**
A:`fleet hosts enroll-script` / 面板加主机会返回 reason
`license.trial_expired`;已接入主机的部署 / 管理 / 监控全部照常。把购买的
license 文件放到面板机 `/opt/fleet/panel/data/fleetpanel.license` 后
`systemctl restart fleet-panel` 即解除。

**Q:用户想用域名 + HTTPS 访问面板?**
A:面板自身默认 http + 随机入口路径。要域名 https 就在面板机上加反代
(或后续用面板自己的 Sites 功能),改 `/opt/fleet/panel/config.yaml` 的
`http.external_url` 后重启;CLI 重新 `fleet auth login --panel https://…`。

## 禁区

- 管理员密码 / 入口路径:不进仓库、不进项目配置(`~/.fleet` / `.fleet/state.yaml`)、不进日志;
  **只在最终交付块出现一次**。
- 不默认开 22 端口(UserData / 云助手不需要 SSH)。
- 买机部分的全部禁区同 [`host-buy-aliyun`](host-buy-aliyun.md)(AccessKey
  不出本机、不重复下单、CancelOrder 只在用户明说时)。
