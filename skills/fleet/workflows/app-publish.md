---
id: app-publish
title: 从代码到 push 上线(主线)
inputs: [project_dir]
sub_workflows: [cert-enable-https, host-pick, host-enroll, dns-account-add, acme-account-add, source-pick, expose-domain]
---

# app-publish

主线。从「帮我发布」/「push 上去」到容器在远端健康跑起来。

## Step 1 · 检测项目骨架

```sh
fleet projects which --output json
```

- exit 0(已解析到项目,来源 registry 或 local 都算)→ 跳 Step 4
- 非 0(当前目录还不是登记的 fleet 项目)→ 继续 Step 2

> ⚠️ 用 `fleet projects which` 判断,**不要** `test -f .fleet/state.yaml`:
> 默认 `fleet init` 写的是 `~/.fleet` 注册表,项目根本没有那个文件。

## Step 2 · 解析 host

```sh
fleet hosts list --output json
```

按 `conventions.md::host 选择(D21)`:
- 用户已在对话里指了 → 校验存在 + online → 写到 init args
- panel 只有 1 台 online → 自动用
- ≥2 台 online → **inline** 列让用户选,不替决定
- 0 台 online → 套 [`host-enroll`](host-enroll.md)

## Step 3 · `fleet init`

```sh
fleet init --app <app> --hosts <host1>,<host2> --output json
```

**默认登记到 `~/.fleet` 注册表**(按目录绑定,不污染源码仓)。要团队共享、随
仓库走时加 `--local` 写一份可提交的 `.fleet/state.yaml`。`--reset` 仅在用户
明确说"重来"才加。

## Step 4 · 决定版本号(D20)

```sh
fleet app-versions latest <app> --output json
```

按 `conventions.md::镜像版本号(D20)` 决定 vX.Y.Z:

1. 用户指 → 照用
2. CLAUDE.md / AGENTS.md bump 规则 → 套
3. 解析到的项目的 `version_policy` → 套(semver = patch +1 / minor / major)
4. 都无 → 默认 semver patch +1

panel 三校验失败的 reason:
- `version.malformed` → AI 修格式重试
- `version.duplicate` → AI bump +1 重试
- `version.not_monotonic` → AI bump 到 `latest_pushed +1` 重试

## Step 5 · `fleet preflight`

```sh
fleet preflight <app> --output json
```

- exit 0 → 进 Step 6
- exit 2(`ready=false`):按 `gaps[].resolve` 套 sub-workflow 顺序补
  (resolve id 与 `workflows/<id>.md` 一一对应,服务端 preflight_test.go 钉死):
  - `host-pick`(state.hosts 空 / 与 panel instance 分布 drift)
  - `host-enroll` / `source-pick` / `expose-domain` /
    `cert-enable-https` / `dns-account-add` / `acme-account-add`
  - 补完**重跑** preflight,直到 ready=true 或同一 gap 出现两次(死循环熔断,exit 2 给用户)
- exit 1 → 引导用户先修 token / panel 离线

注意 `instances` 维度(drift 检查):push 作用于 **panel 上该 app 的全部
instance**,不是项目声明的 hosts。两边不一致时 preflight 会报
`drift` gap —— 必须先按 `host-pick` 对齐再 push,否则体检结论对不上 push
实际触碰的机器。

## Step 6 · push + 流式观察

> **分叉:`state.source=static`(纯前端 / 静态站点)**
>
> 没有容器版本、没有 D20 三校验、不要 image registry / buildx。流程退化成:
>
> 1. 检查 `state.static_dir`(默认 `./dist`)非空。
> 2. `fleet push <app> --watch` —— 不传 `--version`,CLI 自动把目录打 tar.gz
>    上传到 `/apps/{app}/site:upload`。站点不存在时 panel 用项目的
>    `expose.domain` / `hosts[0]` **自动创建**静态站点(幂等,D3)——
>    不需要先去 UI 建站。`--watch` 流式跟完解包 3 步,exit code 可信。
> 3. 错误码:`site.not_found`(无 domain 可自动创建,补 `fleet init --update
>    --domain` 或去 UI 建)/ `site.not_static`(同名 site 类型错了,操作者
>    介入)/ `site.payload_too_large`(超 1 GiB)。
>
> 静态站点不进 Step 7 的 `fleet status`(那是给容器 app 的)。**验收用
> `fleet probe https://<state.expose.domain> --output json`** —— reachable +
> 2xx/3xx 即上线完成,不要让用户自己开浏览器确认。

```sh
fleet push <app> --version vX.Y.Z --watch --output json
```

push 默认**非阻塞**(202 受理即返回;rollout / 版本终态 / 锁释放全在服务
端)。AI 编排**始终加 `--watch`**:逐 instance 流式跟到终态,任何一台失败
exit code 非 0(终态取自 `plan_finished` 事件,exit code 可信,不需要再去
翻 final_state)。如果会话中断 / 没跟上,用 `fleet push status <app>`(持
久化,panel 重启后仍可查)或 `fleet push watch <app>` 接回。

- exit 0 → 全部 host 滚完,进 Step 7
- exit 2 reason `push.no_instances` → 套 `host-enroll` 后 `fleet app-instances create`
- exit 2 reason `push.preflight_failed` → 服务端硬前置没过(host 离线 /
  source 无效),跑 `fleet preflight` 拿细节按 Step 5 处理
- exit 1 部分/全部 host 失败 → 错误信息标明哪台挂在哪步;套
  [`troubleshoot-build-failed`](troubleshoot-build-failed.md) 或
  [`troubleshoot-runtime-failed`](troubleshoot-runtime-failed.md)。
  失败的版本号**不被占用**,修好后可同版本号重推(见 FAQ)

## Step 7 · 健康确认

```sh
fleet status <app> --wait 30s --metrics --output json
fleet app-versions latest <app> --output json
```

判据(两个都要):

- `app-versions latest` 的 `active == vX.Y.Z`(刚 push 的版本)——active 只在
  **所有** rollout 成功后才切换,这是「真的上线了」的权威信号
- `status` 各 host `cpu_pct/mem_pct` 非空(容器活着在跑)

满足 → 报"上线完成 vX.Y.Z";`active` 还是旧版本 → push 没有全成,看
`fleet exec-actions list --app <app>` 找失败的 action;exit 2 reason
`status.never_healthy` → 套 `troubleshoot-runtime-failed`

## 资源锁退避

按 `conventions.md::资源锁(D18)`:`409 resource.locked` CLI 自动退避 ≤3 次,workflow 不感知。

## FAQ

**Q:panel 说 `version.duplicate`,但我从没成功 push 过 v1.2.0?**
A:先 `fleet app-versions latest <app> --output json` 看
`latest_pushed_status`:
- `failed` —— 失败的版本号不被占用,**直接同版本号重推**(panel 对 failed
  build 豁免 duplicate/monotonic 校验);
- 其他状态(building / finished)—— 这个号真的被用过(或还在跑),bump 到
  下一格;`building` 时先等当前 push 到终态(app 锁也会挡住并发 push)。

**Q:`preflight` 一直 `cert.host_unresolved`,我 install_on 写了 prod-1 但报 host not enrolled?**
A:项目的 `hosts` 里写的是 panel 显示的 `hostname`,不是 IP。跑
`fleet hosts list --output json`,用 `fleet init --update --hosts <label>` 把 `label` 字段填回项目。

**Q:`push` exit 0 还需要再确认什么吗?**
A:看有没有 `--watch`。`push --watch` 的 exit code 可信:逐 rollout 流式跟
进,任何一台失败都是非 0,失败信息带 host + 失败步骤;exit 0 后走 Step 7
双判据(`active == vX.Y.Z` + metrics 非空)做最终确认即可。**不带 `--watch`
的 push exit 0 只代表「已受理」**,终态要用 `fleet push status <app> --wait`
或 `fleet push watch <app>` 拿。

**Q:能跳过 preflight 直接 push 吗?快一点。**
A:跳不掉 —— 服务端在 push 内部**强制**跑硬前置(D14:以 panel 实际
instance host 集合查 host 在线 + source 有效),失败返
`push.preflight_failed`,不进入 rollout。CLI 侧的 `fleet preflight` 额外查
cert / dns / acme / drift(这些需要项目里的 tls 意图,服务端拿不到),
所以 Step 5 仍然要跑。preflight 全并发,3 个 host 也就 1s 左右。

**Q:多 host 部署中第二台失败了,前面那台已经切了新版本,要回滚吗?**
A:push 的错误信息会标明哪台失败。注意此时 `app-versions latest` 的
`active` **不会**切到新版本(active 只在全部成功后翻转),但成功的 host 上
容器已经在跑新 image。处置:**先在失败 host 上单独排障**(套
`troubleshoot-runtime-failed`),修好后**同版本号重推**(失败 push 的版本号
不被占用,重推对已成功的 host 是幂等重滚);不要全集群回滚。
