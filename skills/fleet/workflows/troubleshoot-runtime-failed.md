---
id: troubleshoot-runtime-failed
title: 容器跑挂了 / 起不来
entry_step: fleet doctor
---

# troubleshoot-runtime-failed

覆盖 d3(CI silent)/ d5(rollback)/ d6(boot fail)的诊断路径。

## Step 1 · doctor

```sh
fleet doctor <app> --output json
```

## Step 2 · 拉日志

```sh
fleet logs <app> --tail 200 --level error --output json
```

返 `[{at, instance, message}, ...]`。AI 扫 message 找 panic / fatal /
`bind: address already in use` / `cannot connect` 等模式。

按发现:
- DB 连不上 → 检查 `fleet env list <app>`,看 DB_URL 是不是缺/错
- 端口冲突 → 让用户检查项目的 `expose.port`,或换端口
- secret 漏 → 套 env 配置(可能要套 ci-wire-github 重发 secret)

## Step 3 · 不挂但表现差

```sh
fleet status <app> --metrics --output json
```

`cpu_pct > 90` 持续 → 提扩容:

```sh
fleet app-instances scale <id> --replicas $((current+1)) --output json
```

`mem_pct > 90` → 检查 resources.memory_limit,或扩容。

## Step 4 · 灰度回滚

修不动 + 用户压力大 → 直接回滚:

```
现在最快的止血是回到上一个版本。我帮你滚一次?(Y/n)
```

Y → 套 [`rollback`](rollback.md)。

## Step 5 · 回到主线

止血后引导用户:
1. 真因写测试复现
2. fix → push 新 patch 版本

修完后回到调用方 workflow。

## FAQ

**Q:`fleet logs` 返空,但容器确实跑挂了?**
A:容器可能已经 exited(`docker ps -a` 才能看到)。logs 命令拉的是 `app-<name>-1`
活容器;如果 panel 端 `desired_state=running` 但 host 上 docker 已经退出多次,
agent 会 wait + restart。升级路径:先 **不用 SSH** 拉主机 journald 看 host 层
(docker daemon / agent / 内核 OOM):

```sh
fleet logs host <host_id> --priority 0-4 --since -30m --output json
```

再 `fleet exec-actions list` 看 deploy 历史;还定位不到容器退出原因(容器
stdout 不进 journald)才让用户 SSH 上去
`docker ps -a | grep <app>` + `docker logs <exited_id>`。

**Q:`scale --replicas 2 → 4` 后,cpu 还是 100%?**
A:容器单核 cpu limit 没改。`fleet app-instances get <id>` 看 `resources.cpu_limit`;
单容器吃满才是 100%,扩 replica 是横向 —— 流量没分摊到新 replica 上(OpenResty
的 upstream 还没切)→ 看 `fleet status --metrics` 是不是 hosts list 里只有一个
host 的 cpu 高。

**Q:`docker stats` 数字看着不对,跟 host 自身 `top` 不一致?**
A:`docker stats` 的 CPU% 是按 host 总核数算的(单核满 = 100/N% 不是 100%)。
status 接口透传原始百分比;AI 给用户解读时要乘以核数。

**Q:回滚后还是失败,doctor 也说不出原因?**
A:多数是**外部依赖**:DB 挂了 / Redis 没响应 / 上游 API down。doctor 范围只
看 panel 直接管的 host + container。让用户 SSH 进 host 跑
`docker exec <container> nc -zv <db_host> <port>` 测外部依赖。

**Q:容器 OOMKilled,但 `mem_pct` 看着不高?**
A:`docker stats` 给的是**当前瞬时**值,OOMKill 是峰值触发的。先
`fleet logs host <host_id> --priority 0-4 --since -1h` 扫内核 OOM killer 记录
(免 SSH),要确证再看
`docker inspect <id> --format '{{.State.OOMKilled}} {{.RestartCount}}'`。修法是
升 `resources.memory_limit` —— CLI **没有** `app-instances update`,去 panel UI
实例详情改 spec(只改 replicas / host 分布才走 `scale`)。
