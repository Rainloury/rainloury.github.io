---
title: "自建 Tailscale DERP 中继：国内网络下把打洞变成可控"
date: 2026-03-15
tags: ["Tailscale", "DERP", "内网穿透", "VPS"]
location: 武汉
---

Tailscale 默认用的是官方公共 DERP 节点，在国内经常慢、偶尔连不上。我有一台阿里云国内 ECS（1.6G 内存，BT 面板 6688），干脆自建一个 DERP 中继，把"打洞靠运气"变成"中继我自己管"。

## 为什么自建

- 公共 DERP 节点绕路到境外，延迟高。
- 自建中继走国内 ECS，手机/笔记本回家里的 NAS 延迟低、稳。
- 1.6G 小内存的 ECS 跑一个 derper 绰绰有余。

## DERP 中继的部署

在一台有公网 IP 的机器上跑 `derper`，对外用 TCP 8443（因为 443 被占用/受限）。

```yaml
# docker-compose 片段（占位，真实证书见下方说明）
services:
  derper:
    image: fredliang/derper:latest
    container_name: derper
    restart: always
    command:
      - /derper
      - --hostname=aliyun-cn
      - --addr=:8443
      - --verify-clients
      - --private-key=/cert/derper.key
    ports:
      - "8443:8443"
    volumes:
      - ./derper:/cert
```

关键点：

- `--verify-clients`：只允许登录了你 Tailnet 的节点使用，避免被白嫖当公共中继。
- `--hostname=aliyun-cn`：这个 hostname 会作为 Region 名出现在 Tailscale 里。

## 让节点只用自建中继

在 tailscaled 启动参数里关掉默认区域，只留自建：

```
--advertise-tags=tag:derp
OmitDefaultRegions=true
```

自建 DERP 的 region 配置（RegionID 用 `901` 这类自选编号，别和官方撞）：

```
"Regions": {
  "901": {
    "RegionID": 901,
    "RegionCode": "aliyun-cn",
    "Nodes": [{
      "Name": "aliyun-cn",
      "RegionID": 901,
      "HostName": "YOUR_VPS_PUBLIC_IP",
      "DERPPort": 8443
    }]
  }
}
```

设了 `OmitDefaultRegions=true` 之后，所有 Tailscale 流量都走你这台 `aliyun-cn`，可控、可观测。

## 顺带：RustDesk 也走这台中继的思路

RustDesk 的外网我放弃了 dns01 + Caddy 那套（证书/DNS 太折腾），改成 **IP 直连 + InsecureForTests 模式走 TCP 8443**。本质和 DERP 一样：自建中继、非标端口、不依赖被封的 443。

## 小结

自建 DERP 的投入很小（一台最便宜的国内 ECS），回报很大——Tailscale 从"看天吃饭"变成"自己说了算"。唯一要记得的是 **verify-clients 一定要开**，否则你的中继会变成全网公共节点。

> ECS 公网 IP、RegionID、证书路径以你自己的运维 bundle 为准；本文给的是架构与关键参数。
