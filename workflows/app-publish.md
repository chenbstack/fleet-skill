---
id: app-publish
title: 从代码到 push 上线(主线)
inputs: [project_dir]
sub_workflows: [cert-enable-https, host-enroll, dns-account-add, acme-account-add]
---

# app-publish

主线。从「帮我发布」/「push 上去」到容器在远端健康跑起来。

## Step 1 · 检测项目骨架

```sh
test -f .fleet/state.yaml
```

- 存在 → 跳 Step 4
- 不存在 → 继续 Step 2

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
fleet init --hosts <host1>,<host2> --output json
```

写 `.fleet/state.yaml`。`--reset` 仅在用户明确说"重来"才加。

## Step 4 · 决定版本号(D20)

```sh
fleet app-versions latest <app> --output json
```

按 `conventions.md::镜像版本号(D20)` 决定 vX.Y.Z:

1. 用户指 → 照用
2. CLAUDE.md / AGENTS.md bump 规则 → 套
3. `state.yaml::version_policy` → 套(semver = patch +1 / minor / major)
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
- exit 2(`ready=false`):按 `gaps[].resolve` 套 sub-workflow 顺序补:
  - `cert-enable-https` / `dns-account-add` / `acme-account-add` /
    `host-enroll` / `image-registry-add` / `source-pick`
  - 补完**重跑** preflight,直到 ready=true 或同一 gap 出现两次(死循环熔断,exit 2 给用户)
- exit 1 → 引导用户先修 token / panel 离线

## Step 6 · push + 流式观察

> **分叉:`state.source=static`(纯前端 / 静态站点)**
>
> 没有容器版本、没有 D20 三校验、不要 image registry / buildx。流程退化成:
>
> 1. 确认 panel UI 已创建同名 **静态站点**(`type=static`,name=`<app>`)。CLI
>    暂不支持 sites CRUD,这一步走 UI。
> 2. 检查 `state.static_dir`(默认 `./dist`)非空。
> 3. `fleet push <app>` —— 不传 `--version`,CLI 自动把目录打 tar.gz 上传到
>    `/apps/{app}/site:upload`,返 `{action_id, total_steps}`。
> 4. `fleet exec-actions watch <action_id>` 流式跟解包 3 步:`find -delete` →
>    `tar -xzf` → `rm tmp`。
> 5. 错误码:`site.not_found`(app 名没对上)/ `site.not_static`(类型错了)
>    / `site.payload_too_large`(超 1 GiB);三个都需要操作者介入。
>
> 静态站点不进 Step 7 健康确认 —— `fleet status` 是给容器 app 的。
> 验收方式:浏览器访问 `state.expose.domain`。

```sh
fleet push <app> --version vX.Y.Z --output json
```

返 `{action_id, total_steps}`,立即:

```sh
fleet exec-actions watch <action_id>
```

- exit 0 → 进 Step 7
- exit 2 reason `push.no_instances` → 套 `host-enroll` 后 `fleet app-instances create`
- exit 1 中途失败 → 套 [`troubleshoot-build-failed`](troubleshoot-build-failed.md)
  或 [`troubleshoot-runtime-failed`](troubleshoot-runtime-failed.md)

## Step 7 · 健康确认

```sh
fleet status <app> --wait 30s --metrics --output json
```

- `deployed_at` 已更新 + 所有 host `cpu_pct/mem_pct` 非空 → 报"上线完成 vX.Y.Z"
- exit 2 reason `status.never_healthy` → 套 `troubleshoot-runtime-failed`

## 资源锁退避

按 `conventions.md::资源锁(D18)`:`409 resource.locked` CLI 自动退避 ≤3 次,workflow 不感知。

## FAQ

**Q:panel 说 `version.duplicate`,但我从没 push 过 v1.2.0?**
A:上一次 push 失败前 `AppVersion` 行可能已经写进去了(D20 校验后才 push)。
查 `fleet app-versions list <app>` 确认,直接 bump 到下一格。

**Q:`preflight` 一直 `cert.host_unresolved`,我 install_on 写了 prod-1 但报 host not enrolled?**
A:`state.yaml::hosts` 里写的是 panel 显示的 `hostname`,不是 IP。跑
`fleet hosts list --output json`,把 `label` 字段填回 state.yaml。

**Q:`push` 命令 exit 0 了,但 `fleet status` 还是旧 `active_version`?**
A:push 返 `action_id` 后立即返(rolling 部署是异步的)。必须接
`fleet exec-actions watch <action_id>`,直到 `final_state=succeeded`。中途失败
但 push 命令本身已经 exit,所以 exit code 不能直接信。

**Q:能跳过 preflight 直接 push 吗?快一点。**
A:不能跳过。`push` 内部也跑 D20 三校验(form / duplicate / monotonic)。
preflight 还多查 cert / dns_account / host,不跑 preflight 多数 push 也会因
`push.no_instances` 或 `version.not_monotonic` 失败。preflight 全并发,3 个 host
也就 1s 左右,真不值得跳。

**Q:多 host 部署中第二台失败了,前面那台已经切了新版本,要回滚吗?**
A:看 `fleet status` —— 已 deploy 的 host 上 `active_version` 已更新,失败的还在
旧版本。**先在失败 host 上单独 rollback**(`fleet app-instances rollback
<failed_id>`),再修原因 + push 同版本号;不要全集群回滚。
