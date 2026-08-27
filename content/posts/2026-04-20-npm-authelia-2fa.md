---
title: "NPM + Authelia：给公网服务加 TOTP 二步验证与 DNS-01 证书"
date: 2026-04-20
tags: ["NPM", "Authelia", "HTTPS", "2FA", "公网访问"]
location: 武汉
---

把 NAS 服务暴露到公网（哪怕走非标端口），第一步绝不是"直接开端口"，而是**反代 + HTTPS + 二步验证**三件套。我这套用 Nginx Proxy Manager（NPM）做反代与证书，Authelia 做 TOTP 前置校验。

## 整体链路

```
外网 https://*.example.com:100
  → 路由器 100 → 192.168.1.100:1100 (NPM 容器内 443)
  → NPM 按域名反代到内网服务
  → 受保护的服务前先过 Authelia 校验 TOTP
```

对外只开 100 这一个端口，其余服务全在内网，不裸奔。

## 证书：为什么用 DNS-01

运营商封了 80/443，意味着 HTTP-01 校验（需要 80 端口回源）走不通。所以证书签发必须走 **DNS-01**——在 DNS 服务商那边加一条 TXT 记录来证明域名所有权，完全不依赖入站端口。

NPM 里选"Let's Encrypt + AliDNS"（阿里云），填 RAM 子账号的 **AccessKey / Secret**（仅 DNS 权限，最小授权），NPM 会自动用 AliDNS 完成 DNS-01 校验并签发证书。

## Authelia：公网服务的守门员

Authelia 负责在 NPM 反代**之前**拦一道：未登录/未过 TOTP 的人进不来。

`configuration.yml` 关键段（敏感值用占位符）：

```yaml
session:
  secret: <随机长字符串>
  expiration: 1h
totp:
  issuer: homelab
authentication_backend:
  file:
    path: /config/users_database.yml   # 用户与 argon2 密码哈希
access_control:
  rules:
    - domain: "*.example.com:100"
      policy: one_factor        # 或 two_factor 强制 TOTP
```

`users_database.yml` 里密码用 **argon2 哈希**，别存明文。生成哈希用 Authelia 自带命令：

```bash
authelia hashes generate argon2 <你的密码>
```

## NPM 里挂上 Authelia

在 NPM 的 Proxy Host → Advanced 里，加一段前置校验（片段，占位）：

```nginx
location / {
    proxy_pass http://<内网服务>:PORT;
}
# Authelia 校验（示意，具体以官方 snippet 为准）
auth_request off;
# ... 省略标准 authelia-location 与 auth_request 配置 ...
```

关键点：**先配好 Authelia 的 location 与 auth_request，再配业务反代**，顺序反了容易把所有请求都拦死或都放行。

## 收尾清单

- [ ] NPM 证书用 DNS-01（AliDNS），别用 HTTP-01
- [ ] 对外只暴露 100，内网服务不直连
- [ ] Authelia 开 two_factor（TOTP），用户密码用 argon2
- [ ] 每次新增服务，NPM 反代 + Authelia 规则 + Homepage 卡片三处一起改

> AK/SK、session secret、用户库密码等敏感项均用占位符；真实配置存我自己的运维 bundle。切勿把 RAM 主账号 AK 用在 NPM。
