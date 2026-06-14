# 🆓 免费网页部署（带数据库、带登录、自动更新）

> 🔍 关键词：`免费网页部署` `网站上线` `0元建站` `nodejs部署` `全栈托管` `render` `supabase` `不花钱建站`

## 一、这方案是什么

简单说：**把你的网站免费放到互联网上，不用买服务器，不用备案，数据不会丢，改了代码自动更新。**

用三个免费服务拼成一台"零元服务器"：Render（运行代码）+ Supabase（存数据）+ GitHub Actions（自动干活）。

它不是 GitHub Pages 那种"只能放静态页面"的玩具——它能跑真实的 Node.js 代码、读写数据库、处理用户登录。

---

## 二、面向什么需求

| 场景 | 说明 |
|------|------|
| 🧪 **原型验证** | 有后端逻辑 + 数据库，不想花钱买服务器 |
| 👥 **内部工具** | 团队 < 100 人，日请求 < 10 万 |
| 🎓 **学习/演示** | 需要真实环境展示 |
| 🔑 **多用户独立 Key** | 每人配自己 API Key，服务端安全转发 |
| 🇨🇳 **免备案** | GitHub 托管代码，境外部署 |

---

## 三、解决什么问题

这套方案本质上替代了以前"花 ¥50-100/月买 VPS + 自己配 Nginx + 配 HTTPS + 配 MySQL + 写部署脚本"的流程。把所有运维复杂度转嫁给了三个免费平台。

对比市面上的选择：

```
GitHub Pages  → 静态行，但你的后端代码一行都跑不了
Vercel        → 前端强，但 Serverless 函数限制太多（超时 10s，无 WebSocket）
Fly.io        → 曾经免费，现在只给 $5 试用，之后必须付费
Railway       → 需要信用卡，免费额度不够 24/7
阿里云 VPS    → ¥50-100/月 + 自己配一切

本方案        → $0/月，自动部署，自动保活，数据库持久化
```

---

## 四、有什么优点

1. **真·全栈** — Node.js 完整生态，不像 Serverless 各种限制
2. **数据不丢** — Supabase PostgreSQL 持久化，不像 Render 免费 tier 磁盘是临时的
3. **防休眠** — GitHub Actions 每 14 分钟心跳，不会 30 秒冷启动
4. **Git push 自动上线** — 不用每次去 Render 点 Sync
5. **安全** — API Key 服务端存储，密码哈希，多用户隔离
6. **可升级** — 用户量上来后，Render $7/mo + Supabase $25/mo 就能接住

---

## 快速开始

1. **复制模板文件**到你的项目
2. 改 `render.yaml` 里的 `name` 和 `keep-alive.yml` 里的 URL
3. 确保你的服务端有 `/api/health` 端点
4. 创建 Supabase 项目 → 获取 `DATABASE_URL`
5. Render 创建服务 → 添加 `DATABASE_URL` 环境变量
6. `git push` → 自动上线

详细步骤见 [SKILL.md](./SKILL.md) 和 [DEPLOY.md](./DEPLOY.md)。

---

## 文件说明

```
├── render.yaml                    # Render Blueprint 配置
├── .github/workflows/
│   ├── keep-alive.yml             # 每14分钟防休眠
│   └── deploy.yml                 # Git push 自动部署
├── SKILL.md                       # 操作步骤配方
├── DEPLOY.md                      # 架构详解 + 对比 + FAQ
└── README.md                      # 本文件
```

---

## 成本

| 服务 | 免费额度 | 费用 |
|------|---------|------|
| Render Web Service | 750 hr/mo, 512MB | $0 |
| Supabase PostgreSQL | 500MB | $0 |
| GitHub Actions | 2000 min/mo | $0 |
| **合计** | | **$0/月** |
