---
id: acme-account-add
title: 注册 ACME 账户(Let's Encrypt 等)
inputs: [email]
---

# acme-account-add

## Step 1 · 已有?

```sh
fleet acme-accounts list --output json
```

非空 → workflow 结束(已有 account 就够用,1 个 panel 全局共享)。

## Step 2 · 收 email

inline 问用户的注册邮箱(LE 续期提醒会发到这里):

```
ACME / Let's Encrypt 需要一个联系邮箱,会用来发证书即将过期的提醒。
请告诉我 email,我帮你注册(只用一次,后续 panel 自管)。
```

## Step 3 · 注册

```sh
fleet acme-accounts add \
  --email <email> \
  --directory letsencrypt \
  --output json
```

`--directory` 默认 letsencrypt(其它候选:zerossl / google-trust),没特别要求别动。

- exit 0 → 完成
- exit 1 → panel 连不到 ACME directory(网络问题),告诉用户后退出

## Step 4 · 验证

```sh
fleet acme-accounts list --output json
```

新行可见 → 完成。

## FAQ

**Q:已经有一个 LE account 了,要不要再加一个生产专用的?**
A:多数情况下不用,一个 panel 一个 account 就够。多加 account 的真正理由是
**LE 主账户 rate limit 是按 account 算的(50 cert/week/account)**,大规模签发
+ 测试环境吃同一个 account 容易撞;这种场景下 prod / staging 各一个 account。

**Q:Step 3 报 `acme: failed to create account: connect: connection refused`?**
A:panel 出向被 firewall 挡了。需要允许 `acme-v02.api.letsencrypt.org:443`
(LE 生产 directory)和 `acme-staging-v02.api.letsencrypt.org:443`(staging)。
ZeroSSL 走 `acme.zerossl.com`。

**Q:能用 ZeroSSL 吗?**
A:可以。`--directory zerossl`。但 ZeroSSL 要 EAB(External Account Binding)
key,目前 CLI flag 还没暴露 `--eab-kid` / `--eab-hmac` —— 临时方案是在 panel
UI 上加,然后 CLI 调一切就用现成 account。

**Q:把 ACME account 删了,所有用它签的 cert 怎么办?**
A:旧 cert 继续有效到 not_after。但 renew 会失败(找不到 account)。删之前
先把所有 cert 都 reissue 到新 account,或者打算手动 renew。
