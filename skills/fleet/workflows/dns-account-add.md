---
id: dns-account-add
title: 接入 DNS 服务商凭据(Cloudflare / Aliyun / …)
inputs: [domain]
---

# dns-account-add

## Step 1 · 识别 provider

inline 问用户(provider list 暂硬编码,后续随 panel 同步):

```
请选 DNS 服务商:
[1] cloudflare    [2] aliyun      [3] dnspod
[4] route53       [5] gcp         [6] 其它(我帮你查 panel 支持的)
```

## Step 2 · 收凭据

按 provider 指定字段(CLI 文档 / panel 文档已列):

- cloudflare:`api_token`(scope: Zone:Read + DNS:Edit)
- aliyun:`access_key_id` + `access_key_secret`
- ... 同上

inline 提示用户在哪拿:**优先给 CLI 命令(`gh secret` 不适用,但 `op read` 类 1Password CLI 适用)**;
没 CLI 退回浏览器链接(`user_actions_required.url_template`)。

## Step 3 · 添加

```sh
fleet dns-accounts add \
  --provider <cloudflare|...> \
  --label "<human-readable>" \
  --credential api_token=$CF_TOKEN \
  --output json
```

返 `{id, provider, label}`。

## Step 4 · 验证

```sh
fleet dns-accounts list --for <domain> --output json
```

回包里能查到这个 account.id → 完成。否则告诉用户凭据可能没覆盖目标 zone。

## FAQ

**Q:Cloudflare token 给了 `Zone:Read + DNS:Edit`,签 cert 还是 403?**
A:90% 是 `Zone Resources` 漏选了具体 zone。CF 默认是 `Include - All zones`,
但用户为了安全很可能选了 `Include - Specific zone - foo.com`,漏勾了实际域名所在
zone。检查方法:Cloudflare dashboard → My Profile → API Tokens → 该 token →
Permissions 看 zone list。

**Q:加了一个 account,改字段(比如轮换 token)怎么办?**
A:panel 不暴露 update —— `delete` 后 `add` 一遍。设计上避免凭据漂移:看到
`dns-accounts list` 里就是当前真值。

**Q:能看凭据明文确认对不对吗?**
A:不能。master key envelope 加密,落库后只能解密给签发流程用。验证凭据对不对
就直接试签一次:`fleet site-certs issue --domain test-$(date +%s).foo.com
--challenge dns-01 --dns-account <id> --output json`(用一次性子域名,签完
立即 delete)。

**Q:Aliyun / DNSPod 凭据格式跟 Cloudflare 不一样?**
A:对。每个 provider 自己一套字段:
- cloudflare:`api_token`
- aliyun:`access_key_id` + `access_key_secret`
- dnspod:`dnspod_token`(`id,key` 形式)
- route53:`aws_access_key_id` + `aws_secret_access_key` + `aws_region`
`--credential` flag 接 k=v,可多次出现,按 provider 填对应字段。
