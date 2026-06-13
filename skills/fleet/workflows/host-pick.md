---
id: host-pick
title: 选定 / 修正项目的 hosts(D21)
when_to_use: |
  preflight 报 gap `host`(项目 hosts 为空)或 `drift`(项目的 hosts
  跟 panel 上实际 instance 分布不一致);或用户想换部署目标机器
inputs:
  - hosts: 目标 host(label 数组,多选时必须用户拍板)
side_effects:
  - 项目的 hosts 字段(`~/.fleet` 注册表或仓内 .fleet/state.yaml,仅本地,不动 panel)
referenced_workflows:
  - host-enroll
---

# 流程

1. **拿事实**:并行跑
   - `fleet hosts list --output json` —— panel 上有哪些 host、谁 online
   - `fleet app-instances list --output json` —— 该 app 的 instance 实际落在哪些 host
2. **分情况**:
   - `drift` gap(两边不一致)→ **inline 问用户哪边是事实**:
     ```
     [1] panel 为准 —— 把 instance 所在 host 写回项目(最常见:别人 scale 过)
     [2] 项目为准 —— 用 fleet app-instances scale 把 instance 挪到项目声明的 host
     ```
     选 1 → `fleet init --update --hosts <instance 实际 hosts>`;
     选 2 → 套 `app-publish` 的 scale 分支(`fleet app-instances scale`)。
   - `host` gap(state.hosts 为空)→ 按 D21 选 host:
     - 用户对话里指了 → 校验存在 + online → `fleet init --update --hosts <label>`
     - panel 恰 1 台 online → 自动用,不打扰用户
     - ≥2 台 online → **inline 多选一**列出(`[1] prod-2 (disk 42G)` …),
       绝不替用户拍板 —— host 决策直接挂钩线上流量打到哪台机器
     - 0 台 online → 套 [`host-enroll`](host-enroll.md)
3. **确认**:重跑 `fleet preflight --output json`,`host` / `drift` 维度应 ok。
