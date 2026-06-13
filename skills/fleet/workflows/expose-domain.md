---
id: expose-domain
title: 补齐 expose.domain(TLS 需要域名)
when_to_use: |
  preflight 报 gap `cert` 且 detail 是 expose.domain unset —— 项目声明
  了 tls(dns-01 / http-01 / auto)却没给域名
inputs:
  - domain: 对外域名(只有用户能答)
side_effects:
  - 项目的 expose.domain(注册表或仓内 state.yaml)
referenced_workflows:
  - cert-enable-https
---

# 流程

1. **域名只有用户能答** —— 直接问:「这个应用准备用什么域名对外?(没有域名
   的话可以改走 ip+port,跳过 HTTPS)」
   - 用户给出域名 → `fleet init --update --domain <d>`
   - 用户说没有域名 → inline 二选一:
     ```
     [1] 改 expose=ip+port,跳过 TLS(fleet init --update,清掉 tls)
     [2] 先去买/解析一个域名,回头再继续
     ```
2. 域名写入后**顺手查证书覆盖**:`fleet site-certs check <d> --output json`,
   没覆盖 → 套 [`cert-enable-https`](cert-enable-https.md)。
3. **确认**:重跑 `fleet preflight --output json`,`cert` 维度应 ok 或转为
   `cert-enable-https` 可解的 gap。
