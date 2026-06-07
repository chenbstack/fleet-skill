---
id: cert-enable-https
title: 给一个域名签发 + 装 HTTPS 证书
inputs: [domain, challenge_type?]
sub_workflows: [dns-account-add, acme-account-add]
---

# cert-enable-https

给 `state.yaml::expose.domain` 配齐 HTTPS。`challenge_type` 没指就按域名挑:
wildcard(`*.foo.com`)只能 dns-01,普通域名优先 http-01。

## Step 1 · 已有 cert 命中?

```sh
fleet site-certs check <domain> --output json
```

返 `{exact, wildcard_match, not_after}`:
- `exact=true` 且 `not_after` >30d → workflow 结束
- `wildcard_match` 命中且未过期 → workflow 结束
- 其它 → 进 Step 2

## Step 2 · ACME / DNS 账号检查

```sh
fleet acme-accounts list --output json
fleet dns-accounts list --for <domain> --output json   # dns-01 才需要
```

- 没 ACME → 套 [`acme-account-add`](acme-account-add.md)
- dns-01 且没匹配 DNS account → 套 [`dns-account-add`](dns-account-add.md)

## Step 3 · 签发

```sh
fleet site-certs issue --domain <domain> \
  --challenge <http-01|dns-01> \
  [--dns-account <id>] \
  [--acme-account <id>] \
  [--install-on host1,host2] \
  --output json
```

`--install-on` 默认取 `state.yaml::hosts`。返新建 cert 行。

- exit 2 reason:
  - `cert.host_unresolved` → install_on 里有未录入主机,套 `host-enroll`
  - `acme.account_required` → 套 `acme-account-add`
- exit 1 → 报错给用户(常见:DNS 凭据没权限改 zone)

## Step 4 · 验证

```sh
fleet site-certs check <domain> --output json
```

`exact=true && not_after` 在未来 → 完成。

## FAQ

**Q:我已经有 `*.foo.com` 的 wildcard,现在加 `api.foo.com`,还要再签吗?**
A:不要。Step 1 的 check 会返 `wildcard_match` 命中,直接跳过签发。但要确认
新加的 site 把 `cert_id` 绑到这个 wildcard cert(panel 端 site→cert 是显式关联,
不是按域名自动)。

**Q:签发成功但浏览器还是 "not secure"?**
A:openresty 没 reload。check 命令看的是 panel 数据库;真正生效要看
`exec-actions watch <issue_action_id>`,确认 cert deploy + reload 步骤都
succeeded 了。如果 deploy 行但 reload fail → SSH 上去 `nginx -t` 看 cert
路径权限。

**Q:dns-01 卡在 "waiting for DNS propagation" >5 分钟?**
A:多数 provider TXT 生效 30s-2m。超过 5 分钟通常是 `dns-account` 配的凭据
没权限改这个 zone(比如 Cloudflare token 漏勾 `Zone:DNS:Edit` 对**这个具体
zone** 的 resource scope)。check 命令:
`dig +short TXT _acme-challenge.<domain>` 看 TXT 有没有真写进去。

**Q:改了 challenge type(http-01 → dns-01)后老 cert 怎么处理?**
A:不会自动改。renew 时还是按签发时的 challenge。要切就 delete 重 issue。

**Q:wildcard 比 exact 安全吗?为啥 check 返结果时两个字段分开?**
A:wildcard 一旦泄露影响整个 subdomain 树,exact 只影响一个域名。check 把
两个分开返,AI 可以按场景选:测试 env 用 wildcard 省事,生产环境敏感
subdomain 单签 exact。
