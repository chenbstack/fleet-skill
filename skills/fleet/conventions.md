# 编排约定

## EE-only 边界

vibe-deploy 的服务端 API(`/api/v1/cli`)是 **EE 专属**。CLI 打到 CE panel
时会返回 reason `panel.no_cliapi` 的结构化错误(而不是裸 404)—— 看到它就
告诉用户:这台 panel 是 CE,vibe deploy 需要 fleetpanel-ee;CE 用户走 panel
UI。不要重试、不要换 flag 绕。

## 项目解析(Hybrid:注册表 + 仓内 state.yaml)

「项目」= 一份配置(panel / app / hosts / expose / source / build /
version_policy)绑定一个目录。CLI 启动时按当前目录解析出当前项目,优先级:

1. 命令行 `--app` / `--panel` 显式覆盖
2. 从 cwd 向上找到的仓内 `.fleet/state.yaml`(团队共享通道:可提交、随仓库
   走、clone 即部署 —— **存在即赢**)
3. `~/.fleet/projects/<name>.yaml` 注册表,按 cwd 做**最长目录前缀匹配**

**默认存储是注册表**:`fleet init` 默认写 `~/.fleet`(按目录绑定,不污染源码
仓、不会把 panel/host 拓扑误推到公开仓);要团队共享时 `fleet init --local`
写一份可提交的 `.fleet/state.yaml`。**真正的 token 只在 `~/.fleet/credentials`
(chmod 0600,从不进仓)**,任何项目配置里都没有 token。

- 判断「当前目录是否已是登记的 fleet 项目」用 `fleet projects which`(解析到则
  exit 0 + 打印来源 local/registry,否则非 0)—— **不要** `test -f
  .fleet/state.yaml`,默认项目根本没有这个文件。
- 一个父目录下平铺多个项目:各自在自己的子目录 `fleet init` 登记即可,靠最长
  前缀匹配区分,互不干扰。
- 查看 / 删除注册项:`fleet projects list` / `fleet projects rm <name>`(rm 只
  删注册表,不动项目目录)。

## Exit code 协议

| code | 含义                                                          |
| ---- | ------------------------------------------------------------- |
| `0`  | 命令完成,继续 workflow 下一步                                 |
| `2`  | 需要用户输入或 gap 需要补,JSON 里 `gaps[*].resolve` 指对应 sub-workflow id |
| `1`  | 命令本身挂了(panel 离线 / token 失效),AI 引导用户先修        |

读 exit code → 决定继续 / 套 sub-workflow / 找用户。**不要靠解析人类
文本判断状态**。长动作命令分两类:
- **跟随类**(`push --watch` / `push watch` / `push status --wait` /
  `exec-actions watch` / components install / site-certs issue)把终态映射
  进 exit code —— exit 0 = 动作真的成功,不需要二次翻 `final_state`。
- **受理类**(不带 `--watch` 的 `push`)exit 0 只表示「已受理」,终态要靠
  跟随类命令或 `push status` 拿。

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
3. 解析到的项目的 `version_policy`(注册表或仓内 state.yaml;`semver` / `external` / `calver`)
4. 都没有 → AI 按默认 semver(patch +1 默认 / 新依赖新 endpoint = minor / 破坏式 = major)

push 前先 `fleet app-versions latest <app>` 拿 panel 端 `latest_pushed` +
`latest_pushed_status` 作参照。Panel 三校验:形式 `^v\d+\.\d+\.\d+$` /
不存在 / 严格大于 latest。失败 reason → AI 自动 bump 或问用户。

**失败的版本号不被占用**:`latest_pushed_status=failed` 时同一 vX.Y.Z 可以
重推(panel 对 failed build 豁免 duplicate/monotonic);`active` 只在所有
rollout 成功后才切换,所以 `active` 是「真的上线了」的权威信号,
`latest_pushed` 只是「最后试过的号」。

## push 服务端硬前置(D14)

push 在服务端**强制**内嵌硬前置:以 panel 上该 app 的实际 instance host
集合(不是项目声明的 hosts)查 host 在线 + source 有效,失败返
`push.preflight_failed`,不进入 rollout —— 剧本绕不过它。意图级前置
(cert / dns / acme / drift)只有 CLI 的 `fleet preflight` 查得到(需要
项目里的 tls 意图),所以 push 前仍然先跑 preflight。

## host 选择(D21)

`fleet init` 必须解析出至少一个 host 写进项目的 `hosts`(必填数组;默认存进
`~/.fleet` 注册表,`--local` 则存仓内 state.yaml):

1. 用户在 `--host` / `--hosts` 明确指了 → 校验存在 + online 写入
2. 没指但 panel 只有 1 台 online → 自动用
3. 没指且 ≥2 台 online → CLI `exit 2 reason: host.required`,SKILL **inline
   多选一** 列 host 给用户(不替用户拍多选)
4. 无 online host → `host.none_online`,引导先 `fleet hosts add`

## 长动作的进度展示(push 非阻塞 + gh 风格命令面)

`fleet push` **默认非阻塞**:202 受理即返回(打印 `rollouts[]`),rollout、
版本终态推进、app 锁释放全在服务端,CLI 断开无影响。命令面对照 gh:

| 要什么                  | 命令                                                |
| ----------------------- | --------------------------------------------------- |
| 推并同步跟完(AI 默认) | `fleet push ... --watch`(一条命令,exit code 可信) |
| 事后查状态              | `fleet push status <app>`(持久化行,panel 重启后仍可查) |
| 事后跟到完成            | `fleet push watch <app>`(重放日志 + 等终态)        |
| 等终态但不要日志        | `fleet push status <app> --wait`                    |
| 单台 rollout 日志       | `fleet exec-actions watch <action_id>`(对已完成 action 重放全量历史) |

**AI 编排默认用 `push --watch`** —— 一条命令拿到全部进度与终态。非阻塞
默认是给人类 / CI 的(发起后做别的事)。其他返回 `{action_id}` 的长动作
(装组件 / 证书签发)用 `fleet exec-actions watch <id>` 流式跟。**不要**
用 sleep 轮询、不要自己写 status poll 循环 —— `--wait` / `watch` 已内置。

注意:ExecAction(日志流)在 panel 内存里,重启后 `action not found`;
但 `push status` 读持久化行,**重启后依然可答**。所以 watch 不上时先
`push status`,再不行才走 app-publish Step 7 的判据(`app-versions
latest` 的 `active` + `fleet status`)。
