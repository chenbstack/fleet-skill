---
id: source-pick
title: 确定项目的 source 类型
when_to_use: |
  preflight 报 gap `source`(项目 source 未设);或 fleet init 时不确定
  项目以什么形态交付
inputs:
  - source: tarball | dockerfile | image | static
side_effects:
  - 项目的 source(及 static_dir / build)字段(注册表或仓内 state.yaml)
---

# 流程

1. **AI 先自己看项目**,大多数情况不用问用户:
   - 有 `Dockerfile` → `dockerfile`
   - 纯前端构建产物(`dist/` / `build/`,无服务端进程)→ `static`
   - CI 已产 image、项目里有 registry 引用 → `image`
   - 都不是但可以打包源码(panel 侧 build)→ `tarball`
2. **判断有歧义时**(比如同时有 Dockerfile 和 dist/)inline 二选一,
   一句话讲清差别,默认推 AI 的判断。
3. **写入**:`fleet init --update --source <s>`;`static` 时顺手确认
   `--static-dir`(默认 `./dist`)。
4. **确认**:重跑 `fleet preflight --output json`,`source` 维度应 ok。
