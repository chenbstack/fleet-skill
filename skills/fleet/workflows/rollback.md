---
id: rollback
title: 回滚一个 instance 到历史 build
inputs: [app, to?]
---

# rollback

## Step 1 · 选回滚目标

用户没指 `--to` → 列最近 versions 让用户选:

```sh
fleet app-versions list <app> --recent 10 --output json
```

inline 列:

```
近 10 次版本(标星的是 active):
[1] v1.6.2 ★ active (3h ago)
[2] v1.6.1   (1d ago)        ← 默认回滚目标(上一个非 rolled_back)
[3] v1.6.0
...
回我 [2] / 直接版本号 / "cancel"
```

`--to` 接受 vX.Y.Z 或 build_id(`b047`)。

## Step 2 · 列要回滚的 instances

```sh
fleet app-instances list --app <app> --output json
```

跨多 host 时全部回滚(不细分 partial)。

## Step 3 · 逐 instance 回滚

```sh
fleet app-instances rollback <instance_id> --to <vX.Y.Z|build_id> --output json
```

返 `{action_id, total_steps}`:

```sh
fleet exec-actions watch <action_id>
```

多 host 串行(不并行 —— 灰度回滚 + AI 能在第一台失败时停)。

- exit 2 reason `rollback.unresolved` → `--to` 不存在,回到 Step 1 重选
- exit 1 → 容器拉不到旧 image(已 GC?)→ 套 `troubleshoot-runtime-failed`

## Step 4 · 验证

```sh
fleet status <app> --wait 30s --output json
```

`active_version` == 回滚目标 → 完成。**panel 已自动给原 active version 标
`rolled_back: true`**,下次 `app-versions latest` 不会推这个版本。

## 提示用户

回滚不会改 git。如果是修线上 hotfix,提醒用户:
> 回滚只回 panel 端 + 容器;代码还在 main 上,记得手工 revert 或挑 cherry-pick。

## FAQ

**Q:回滚到 v1.6.1 成功了,下次 push 还能用 v1.6.2 吗?**
A:不能。v1.6.2 还在 `app-versions list` 里,且作为**回滚源**被标了
`rolled_back: true`(标的是回滚源,不是回滚目标 v1.6.1)。push 同版本号会被
`version.duplicate` 拒;bump 到 v1.6.3 起跳。

**Q:`rollback.unresolved`,我用的版本号 panel 里明明有?**
A:两个常见原因:(1)拼错了,vX.Y.Z 字面比较;(2)那个 version 行没绑
build_id(`external` policy 的 placeholder version)—— 只有有 build_id 的版本
能 rollback。看 `fleet app-versions list <app> --output json`,挑 `build` 字段
非空的。

**Q:image registry 把旧 tag GC 了,回滚拉不到 image?**
A:`exec-actions watch` 里会看到 `docker pull` step 失败。两条路:
- 如果还有源码 sha → checkout 那个 sha + 重新 build + push 新版本(等同前滚)
- 没源码 → 只能从健康的 host 上手动 `docker save / load` 到挂掉的 host,
  然后 panel-side 把 image_pull step 标成已完成

**Q:回滚多 host,第一台失败要不要继续?**
A:**默认不继续**。workflow 串行,第一台 fail 就 break。理由:多数情况下
失败原因是普适的(image 拉不到 / 配置错),让后面台也滚一遍只是再失败一次。
要并行的话用户得明确说"全部强制回滚"。

**Q:能不能直接 SSH 进 host 用 `docker run` 切容器,绕过 panel?**
A:可以但不建议。panel 数据库会一直认为 `active_version` 是 panel 记录的那个,
下次健康检查 / autoupdate 触发时会"修正"回 panel 端版本,把你手动切的 image
冲掉。要这么干就先 panel UI 上把 app 的 `desired_state` 设 `stopped`。
