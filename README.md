# Rainstone · 折腾手记

个人技术博客，基于 **Hugo** 构建，部署到 GitHub Pages 用户站。

## 本地预览

需要 Hugo **extended** 版（≥ 0.165.0）。

```bash
# 安装（任选其一）
#   官网：https://github.com/gohugoio/hugo/releases
#   国内镜像（GitHub 资源 CDN 被墙时）：
#   https://mirrors.tuna.tsinghua.edu.cn/github-release/gohugoio/hugo/LatestRelease/

# 本地起服务（带热刷新）
hugo server -D

# 仅构建产物到 public/
hugo --minify
```

> 如果你本地装不上 Hugo（国内直连 GitHub 资源常超时），**不用管本地**——
> 直接 push 到 `master`，GitHub Actions 会在云端用 Hugo 构建并发布。

## 部署（GitHub Pages + Actions）

1. 仓库 **Settings → Pages → Build and deployment → Source** 选 **GitHub Actions**。
2. 把本目录推送到 `master`：
   ```bash
   git add .
   git commit -m "rebuild: hugo blog"
   git push origin master
   ```
3. Actions 跑完，站点出现在 GitHub Pages 用户站（`https://<你的用户名>.github.io/`）。
4. 之后每次 push 自动重新构建发布。

## 目录结构

```
.
├── config.toml            # 站点标题/描述/菜单/社交
├── content/
│   ├── _index.md          # 首页 hero 文案
│   ├── about.md           # 关于页
│   ├── posts/             # 篇章一：NAS / 家庭 IT 自建
│   └── network/           # 篇章二：网络（远程访问 / 家庭网络排障 / 反向代理）
├── layouts/               # 自写主题（baseof/index/single/list + partials）
├── static/
│   └── css/style.css      # 自适应浅色/深色样式
└── .github/workflows/     # hugo.yml 云端构建发布
```

## 写一篇新文章

```bash
# NAS / 家庭 IT 自建 篇章
hugo new posts/2026-05-01-my-topic.md

# 网络 篇章（远程访问 / 家庭网络排障 / 反向代理，用对应标签区分）
hugo new network/2026-05-01-my-topic.md
```

生成的文件在对应目录下，填好 front matter（`title` / `date` / `tags` / `draft`）和正文即可。
网络篇章里的文章建议带上分组标签：`远程访问` / `家庭网络排障` / `反向代理`，便于在篇章页归类。
草稿设 `draft: true` 时，线上用 `hugo server -D` 才看得到；正式发布改 `false` 再 push。

## 绑定自定义域名（可选）

1. 在 `static/` 下放一个 `CNAME` 文件，内容填你的域名（如 `example.com`）。
2. DNS 里把该域名的 CNAME 指向你的 GitHub Pages 用户站。
3. 仓库 Settings → Pages → Custom domain 填该域名，等校验通过（需 DNS 生效）。

## 安全说明

文章中涉及密码、AccessKey、IP、真实域名、邮箱等一律用占位符，**真实值不入库**。
对外暴露的服务统一过 NPM + Authelia（TOTP），证书用 DNS-01 校验。
