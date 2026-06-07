---
id: troubleshoot-build-failed
title: 编译挂了的诊断回路
entry_step: fleet doctor
---

# troubleshoot-build-failed

固定第一步:`fleet doctor`,以 issues 作分支入口(D19)。覆盖 CI 失败 / panel
本地 build 失败 / 镜像拉取失败三个最高频路径。

## Step 1 · doctor

```sh
fleet doctor <app> --output json
```

返 `{healthy, checks, issues[]}`。下面按 `issues[].id` 分支。

## Step 2 · 分支处理

### `deploy.in_progress`

上一个 push / rollback 还在跑,不是真挂。

```sh
fleet exec-actions list --app <app> --output json
```

找到 `final_state` 为空的 action,流式跟它:

```sh
fleet exec-actions watch <action_id>
```

跟完后 re-run doctor。

### `instance.deploy_failed`

```sh
fleet exec-actions list --app <app> --recent 5 --output json
```

取最新一条 `final_state=failed` 的 `failed_step` 配合 step 的 `stderr_tail`,
AI 直接给用户原因 + 修复建议(常见:Dockerfile syntax / docker login 凭据失效)。

### CI 编译失败(panel doctor 看不到,只能从 ci-events 推)

```sh
fleet ci-events list <app> --recent 5 --output json
```

最近一条 `stage=failed` 的 `logs_tail` + `ci_run_url`:
- 有 GitHub `gh` CLI → `gh run view <run_id> --log-failed` 拉详细 log
- 没 `gh` → 给用户 `ci_run_url` 链接

### `host.offline`

```sh
fleet hosts list --output json
```

确认离线主机 ID。引导用户去 panel UI / 物理排查。

## Step 3 · 修复后重跑

修完后:

```sh
fleet preflight <app> --output json
fleet push <app> --version vX.Y.Z+1 --output json   # bump 一格
```

成功后回到调用方 workflow。

## FAQ

**Q:doctor 全绿但 push 还在挂?**
A:doctor 看的是已部署的**live state**(instance 表 + host 健康)。还在 build
阶段 / push 中途的 step 不在 doctor 范围。改看
`fleet exec-actions list --app <app> --output json` 里 `final_state=failed` 的
最新一条,流式 watch 它的 stderr_tail。

**Q:`exec-actions list` 是空的,但我刚 push 完?**
A:action 是 in-memory,panel 重启就丢。如果 panel 在 push 后重启过,只能复现
一次。push 的 `version.duplicate` 这次没碰到是因为前一次没真正 record(record
在 build 阶段之后);bump 一格再 push 就行。

**Q:CI 失败,`logs_tail` 截断了看不到真错?**
A:`fleet ci-events list <app> --output json` 里有 `ci_run_url`。`gh run view
<run_id> --log-failed` 拉完整 log。没有 gh CLI 就让用户点链接。

**Q:已经 push 5 次都失败,版本号被占完了?**
A:正常。失败的 build 也会写 AppVersion 行(panel 三校验先于 build)。继续 bump
就行,占用的版本不会"释放"。如果用户在意整洁,可以 `fleet app-versions` 后续
加 `--prune-failed`(尚未实现)。

**Q:Dockerfile 没改过,昨天还能 build,今天 `image_pull` 步骤 unauthorized?**
A:registry 凭据可能轮换了。`fleet image-registries list` 看是否还存在;真实
认证测:在 host 上 SSH `docker pull <image>` 看是否同样 unauthorized。如果 host
能拉但 panel push 步骤拉不到,通常是 docker config 不在 agent 的 `$HOME` 路径。
