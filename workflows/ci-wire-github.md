---
id: ci-wire-github
title: 把 fleet push 织进 GitHub Actions
inputs: [repo, app]
---

# ci-wire-github

让用户的 main 分支推送后自动 `fleet push`。仅 GitHub —— GitLab / Drone 思路同。

## Step 1 · 申请 CI token

```sh
fleet tokens create --label "gh-actions-$REPO" --scope admin --app <app> --ttl 365d --output json
```

返 `{id, secret, expires_at}`。**`secret` 仅此一次返回**,workflow 把它喂进 Step 2 后立即丢掉(不写本地)。

## Step 2 · 写 GitHub secret

优先 `gh` CLI:

```sh
echo "$secret" | gh secret set FLEET_TOKEN --repo <repo>
gh secret set FLEET_PANEL --body "$panel_url" --repo <repo>
```

`gh` 不可用 → inline 给链接 + 复制按钮文案(`user_actions_required`):

```
打开 https://github.com/<repo>/settings/secrets/actions
新建 FLEET_TOKEN = <secret>(我已生成)
新建 FLEET_PANEL = <panel_url>
完事回我 "ok"
```

## Step 3 · 写 workflow yaml

新建 `.github/workflows/fleet-push.yml`:

```yaml
name: Fleet Push
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install fleet CLI
        run: curl -fsSL https://fleet.example/install.sh | sh
      - name: Push
        env:
          FLEET_TOKEN: ${{ secrets.FLEET_TOKEN }}
          FLEET_PANEL: ${{ secrets.FLEET_PANEL }}
        run: fleet push <app> --version v${{ github.run_number }}.0.0
```

提醒用户改 `app` 名和版本号策略(`run_number` 默认是 patch++,可换 git tag)。

## Step 4 · 触发一次验证

```sh
gh workflow run fleet-push.yml --repo <repo>
```

之后:

```sh
fleet ci-events list <app> --output json
```

能看到 `build_started` 记录 → CI 接通完成。

## Cleanup 备忘

提醒用户:token 90/365d 后会自动到期。AI 应在用户回到 workflow 时检测
`fleet tokens list --near-expiry 14d`,提前续期。

## FAQ

**Q:`gh secret set` 报 `HTTP 403`?**
A:执行 `gh` 的用户没 repo 的 Settings:write。让 repo owner 跑 Step 2,或者
让用户 inline 走 web UI(workflow 已经准备了 fallback 提示)。

**Q:用 `github.run_number` 当版本号,run 1 → run 2 触发 v1.0.0 → v2.0.0,不太对劲?**
A:对,`run_number` 单调但语义上不是 semver。两个方案:
- 改 `--version v0.${{ github.run_number }}.0`(始终 minor bump)
- 改 trigger 成 `on: push: tags: ['v*']`,用户打 tag 决定版本,CI 用
  `--version ${{ github.ref_name }}`(推荐)

**Q:token 90 天到期,CI 突然挂了怎么办?**
A:`fleet push` 返 401 `auth.token_expired`。AI 应在用户每次回到任何 workflow
时主动跑 `fleet tokens list --near-expiry 14d --output json`,有快到期的就提醒
用户:"GitHub Actions 用的 FLEET_TOKEN 还剩 N 天,要现在轮换吗?"。

**Q:想让 staging / prod 用不同 token + 不同 panel?**
A:GitHub Actions 用 environment-level secrets:
```yaml
jobs:
  deploy:
    environment: prod   # 或 staging
    ...
```
每个 environment 各设自己的 `FLEET_TOKEN` / `FLEET_PANEL`。AI 帮用户建两个
environment,各跑一次 Step 1 issue 不同 scope 的 token(`--app billing-api`
+ `--scope deploy`)。

**Q:CI 已经在跑 docker build,可以让 fleet 直接拉 image 而不是 push 源码?**
A:可以。CI 自己 docker build + push 到 registry 后,workflow 改:
```yaml
- run: fleet push <app> --source image --image $REGISTRY/$APP:v${{ ... }} --version ...
```
panel 不再重新 build,只拉 image 滚动部署。前提:image-registries 里
凭据先配好。
