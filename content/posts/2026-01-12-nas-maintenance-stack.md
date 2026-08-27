---
title: "群晖 DS923+ 运维面板全家桶：一套能长期跑的维护栈"
date: 2026-01-12
tags: ["NAS", "Docker", "运维面板", "群晖"]
location: 武汉
---

家里这台 DS923+ 没核显、4 盘位、4GB ECC（后面会加内存），跑 DSM 7.2 的 Container Manager。为了让它"出事能第一时间知道、平时不用天天盯"，我按**三个 compose 栈**来组织，而不是把所有容器塞进一个文件。

## 为什么要拆成三个栈

- **nas-maintain（维护栈）**：只放"看着机器本身"的工具，允许自动更新。
- **nas-business（业务栈）**：Jellyfin / qBittorrent / Alist / MoviePilot，手动更新，避免自动更搞挂影响看片。
- **nas-tools（工具栈）**：Vaultwarden / File Browser / Apprise 推送 / CookieCloud / RustDesk。

拆开的好处是：维护栈可以放心交给 Watchtower 自动更，业务栈你永远掌控节奏。

## 维护栈六件套 + 辅助

| 服务 | 容器内/宿端口 | 作用 |
|---|---|---|
| Portainer | 9443 | 容器总管理 |
| Netdata | 19999 | 实时资源监控 |
| Scrutiny | 8080 | 硬盘健康（S.M.A.R.T.） |
| Uptime Kuma | 3001 | 存活监控 + 告警 |
| NPM (Nginx Proxy Manager) | 内 1100 → 443 | 反代 + 证书 |
| Homepage | 3000 | 所有服务的总入口 |

辅助：ddns-go（阿里云 DNS 同步公网 IP）、Authelia（TOTP 二步验证）、Redis、Dozzle（日志）。

## Watchtower 只更维护面板

关键决策：**Watchtower 改成标签驱动**，只拉取带特定 label 的容器，业务栈一律不打标签、手动更新。

```yaml
# 维护栈里的 watchtower 片段
services:
  watchtower:
    image: containrrr/watchtower
    environment:
      WATCHTOWER_LABEL_ENABLE: "true"
      WATCHTOWER_POLL_INTERVAL: "3600"
      TZ: Asia/Shanghai
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    labels:
      # 自身也受管理
      com.centurylinklabs.watchtower.enable: "true"
```

每个"允许自动更"的维护面板都在自己的 service 上加：

```yaml
    labels:
      com.centurylinklabs.watchtower.enable: "true"
```

业务栈的容器**不要加这个 label**，Watchtower 就不会碰它们。

## 外网访问：运营商封 80/443 怎么办

我家宽带运营商封了 80/443，所以对外统一走**非标端口 100**：

```
浏览器 → https://*.example.com:100
        → 路由器 100 → 192.168.1.100:1100(NPM 容器内 443)
        → NPM 再按域名反代到各内网服务
```

证书用 **NPM + 阿里云 AliDNS 的 DNS-01** 校验（因为 80 不通，HTTP-01 没法用）。DDNS 由 ddns-go 把公网 IP 同步到阿里云解析。

公网暴露的所有服务前面都过 **Authelia 做 TOTP 前置校验**，不能直接裸奔。

## Homepage 聚合

Homepage 把所有内部服务做成卡片，字段最容易踩坑的是 `apiKey` 和 `key` 不是一回事、`synology` 和 `diskstation` 也不同——每次改配置要同步三处（服务本身、Homepage widget、NPM 反代）。

## 小结

维护栈可以"设完就忘"，业务栈保持手动。先把监控（Netdata + Uptime Kuma）和告警（Apprise / 企业微信）立起来，比什么都重要——**你不需要天天看，但出事它得会叫**。

> 完整 compose 与 Authelia / NPM 片段在我自己的运维 bundle 里（含真实端口与 label 命名），本文只给架构与关键决策。
