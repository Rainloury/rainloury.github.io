---
title: "给 DS923+ 加内存：4GB → 32GB，以及一份 v3 服务的资源账"
date: 2026-02-03
tags: ["NAS", "硬件", "群晖", "DS923+"]
location: 武汉
---

DS923+ 出厂只有 4GB ECC，最高能扩到 32GB。我一开始觉得"够用"，直到上了 MoviePilot v3——然后发现 4GB 根本绷不住。

## 为什么 4GB 不够了

维护栈 + 业务栈 + 工具栈里常驻的容器小二十个，其中最吃内存的是：

- **MoviePilot v3**：强制外接 PostgreSQL + Redis，自己还要起内核浏览器（cloakbrowser），光这一套就吃掉一大块。
- **Jellyfin**：转码虽靠无核显的 CPU 软解（慢但能跑），元数据扫描时也占内存。
- **Netdata / Postgres / Redis**：监控与数据库常驻。

4GB 下，v3 经常触发 OOM 或卡死，DSM 本身也变黏。结论很直接：**这机器该加内存**。

## 资源账（大致，单位 GB，常驻估算）

| 类别 | 占用 |
|---|---|
| DSM 7.2 系统 | ~0.8 |
| 维护栈（Portainer/Netdata/Scrutiny/Uptime Kuma/NPM/Homepage/Redis/Dozzle） | ~1.5 |
| 业务栈（Jellyfin/qBittorrent/Alist/MoviePilot + PG/Redis） | ~4–6（v3 高峰期） |
| 工具栈（Vaultwarden/File Browser/Apprise/CookieCloud/RustDesk） | ~1 |
| **合计峰值** | **~8–10** |

所以 4GB 是绝对不够的，16GB 勉强，32GB 才舒服。我直接上了 32GB。

## 升级步骤（断电操作，别热插拔）

1. 在 DSM 里**正常关机**，拔掉电源与网线。
2. 把机器翻过来，拆掉底盖（两颗螺丝 + 卡扣）。
3. 找到内存插槽，插入 DDR4 SODIMM（ECC 优先，频率按官方兼容表）。
4. 装回底盖，接电开机，进 DSM → 信息中心确认识别到新容量。

## 踩坑提醒

- **ECC 兼容性**：DS923+ 用的是笔记本 ECC SODIMM，不是普通条。买之前先查兼容列表，别贪便宜买错。
- **先备份**：加内存前对重要共享文件夹做一次 Hyper Backup，以防万一。
- **v3 复用维护栈的 PG/Redis**：业务栈的 MoviePilot 不另起 PG/Redis，而是经 `host.docker.internal` 复用维护栈已有的 `postgresql(54320)` / `redis(6379/1)`。这样省内存也省管理，但内存账要一起算。

加完内存后 v3 稳了，DSM 也松快了。这钱花得值。

> PG / Redis 的密码、MoviePilot 的 `SUPERUSER_PASSWORD` 等敏感项用占位符管理，存我自己的运维 bundle，不在此贴出。Redis 密码若含 `@` 在 URL 里要转义为 `%40`。
