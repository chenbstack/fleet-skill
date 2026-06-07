# 编排约定

## Exit code 协议

| code | 含义                                                          |
| ---- | ------------------------------------------------------------- |
| `0`  | 命令完成,继续 workflow 下一步                                 |
| `2`  | 需要用户输入或 gap 需要补,JSON 里 `gaps[*].resolve` 指对应 sub-workflow id |
| `1`  | 命令本身挂了(panel 离线 / token 失效),AI 引导用户先修        |

读 exit code → 决定继续 / 套 sub-workflow / 找用户。**不要靠解析人类
文本判断状态**。

## --output json

所有原子命令支持。AI 永远加 `--output json`,把结果喂给自己判断,不靠
regex 解析终端输出。

## inline 二选一(D6 / D15)

凡是需要用户拍板但选项只有两三个 + 用户大概率选默认,**不开新 turn**,
inline 列:

```
[1] 我帮你 set REDIS_URL 到 panel + 自动重 rollout
[2] 你自己去 panel /apps/billing-api/env 配
```

用户回复 `1` / `2` 即可。**绝不**对 host 选择走 inline 默认 —— D21
要求 panel 多 host 时必须列让用户选。

## `user_actions_required` 解析

某些 gap 需要用户去外部系统点(GitHub secret / DNS 后台改 nameserver),
panel 返 `user_actions_required: [{kind, resolve: ["gh_cli", "github_web"]}]`。
AI 优先用 CLI 实现(`gh_cli` / `glab_cli`),CLI 不可用再回退浏览器链接。

## 资源锁(D18)

panel 写动作按资源粒度做互斥锁(per-instance / per-cert / per-host)。
冲突返 `409 reason: resource.locked` + `Retry-After`。**CLI 自动退避
重试 ≤3 次**(200ms / 500ms / 1200ms),workflow 不用感知。

## 镜像版本号(D20)

`fleet push --version vX.Y.Z` 必填,AI 决定但**用户规则优先**:

1. 用户在对话里直接指了 → 照用
2. 项目根 `CLAUDE.md` / `AGENTS.md` 有 bump 规则 → 套用
3. `.fleet/state.yaml::version_policy`(`semver` / `external` / `calver`)
4. 都没有 → AI 按默认 semver(patch +1 默认 / 新依赖新 endpoint = minor / 破坏式 = major)

push 前先 `fleet app-versions latest <app>` 拿 panel 端 `latest_pushed`
作参照。Panel 三校验:形式 `^v\d+\.\d+\.\d+$` / 不存在 / 严格大于 latest。
失败 reason → AI 自动 bump 或问用户。

## host 选择(D21)

`fleet init` 必须解析出至少一个 host 写进 `state.yaml::hosts`(必填数组):

1. 用户在 `--host` / `--hosts` 明确指了 → 校验存在 + online 写入
2. 没指但 panel 只有 1 台 online → 自动用
3. 没指且 ≥2 台 online → CLI `exit 2 reason: host.required`,SKILL **inline
   多选一** 列 host 给用户(不替用户拍多选)
4. 无 online host → `host.none_online`,引导先 `fleet hosts add`

## 长动作的进度展示

任何长动作(deploy / 装组件 / 证书签发)用 `fleet exec-actions watch <id>` 流式
跟。**不要**用 sleep 轮询、不要用 status 反复 poll。
