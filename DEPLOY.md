# 🏗️ 免费全栈部署方案：Render + Supabase + GitHub Actions

## 一句话概括

**用 Render 跑 Node.js 后端 + Supabase 存数据 + GitHub Actions 防休眠/自动部署，零成本上线一个带数据库、用户系统、API 代理的真实全栈应用。**

---

## 面对什么需求

适合以下场景：

- 🧪 **原型验证 / MVP 测试** — 有完整后端逻辑、需要数据库，但不想花钱买服务器
- 👥 **小团队内部工具** — 用户 < 100，日请求 < 10 万，不需要 24/7 高可用
- 🎓 **学习 / 演示项目** — 需要真实环境展示，但不是商业生产级别
- 🔑 **多用户各自配 Key** — 每人用自己 API Key，服务端只做安全转发
- 🇨🇳 **国内用户** — GitHub 托管代码，Render/Supabase 境外部署，免备案

## 解决什么问题

| 痛点 | 传统方案 | 本方案 |
|------|---------|--------|
| 买服务器贵 | 阿里云/腾讯云最低 ~¥50/月 | **$0/月** |
| Heroku 取消免费 | 被迫迁移 | Render 免费 tier 替代 |
| 文件存储丢数据 | Render 免费 tier 磁盘是临时的 | Supabase PostgreSQL 持久化 |
| 冷启动 30-60s | 无解，只能等 | GitHub Actions 每 14 分钟保活 |
| 每次改代码手动部署 | 手动 scp / FTP | Git push → 自动部署 |
| 数据库要单独配 | 自己装 MySQL/PostgreSQL | Supabase 一键创建，自带管理面板 |
| API Key 暴露前端 | 写死在 JS 里 | 服务端加密存储，每人独立 Key |
| 国内访问 GitHub 慢 | 无解 | SSH 推送（端口 443 免干扰）+ Git 备用 |

---

## 架构一览

```
用户浏览器
    │
    ├─ https://your-app.onrender.com
    │       │
    │       ├─ /                  → 静态前端 (HTML + JS + CSS)
    │       ├─ /api/chat          → 核心 API（代理用户的 AI 调用）
    │       ├─ /api/auth/*        → 用户认证
    │       ├─ /api/settings      → 设置管理
    │       └─ /api/health        → 健康检查
    │
    ├─ Render (免费 tier, Oregon)
    │   ├─ Node.js 20.x
    │   ├─ 512MB RAM
    │   ├─ 750 小时/月 (≈ 24/7)
    │   └─ 15 分钟无活动 → 休眠
    │
    └─ Supabase (免费 tier)
        ├─ PostgreSQL 500MB
        ├─ 自动备份
        └─ 管理面板 (web)
```

### 数据流

```
登录/注册 → 密码哈希 → Supabase users 表
设置保存 → JSON → Supabase user_settings 表 (JSONB)
对话记录 → JSON → Supabase user_conversations 表 (JSONB)
AI 对话  → 用户浏览器 → Render 后端 → AI API
```

### 数据库表结构

```sql
users(email TEXT PK, user_id TEXT, password_hash TEXT, created_at BIGINT)
sessions(token TEXT PK, user_id, email, created_at, expires_at)
user_settings(user_id TEXT PK, data JSONB)
user_conversations(user_id TEXT PK, data JSONB)
```

---

## 有什么优点

### 🆚 vs GitHub Pages

| | GitHub Pages | 本方案 |
|---|---|---|
| 后端代码 | ❌ 不支持 | ✅ 完整 Node.js |
| 数据库 | ❌ 无 | ✅ PostgreSQL |
| 用户登录 | ❌ 无 | ✅ 邮箱+密码 |
| API Key 管理 | ❌ 无法 | ✅ 服务端安全存储 |
| AI 对话 | ❌ 无法 | ✅ 真实 API 调用 |
| 冷启动 | N/A (静态无状态) | 14 分钟心跳保活 |
| 免费 | ✅ | ✅ |

### 🆚 vs Vercel

| | Vercel | 本方案 |
|---|---|---|
| 后端模型 | Serverless (函数) | 常驻进程 |
| 函数超时 | 10-60s (Hobby) | 无限制 |
| WebSocket | ❌ | ✅ (可扩展) |
| Node.js 生态 | 受限 | 完整 npm |
| 数据库 | 需外接 | ✅ Supabase 集成 |
| 商业使用 | ❌ Hobby 禁止 | ✅ 无限制 |

### 🆚 vs 自己买 VPS

| | 阿里云/VPS | 本方案 |
|---|---|---|
| 月费 | ~¥50-100 | $0 |
| 运维 | 手动配置 Nginx/HTTPS/备份 | 平台自动 |
| 自动部署 | 需自己配 CI/CD | Git push 即部署 |
| HTTPS | 需配置证书 | 自动 Let's Encrypt |
| 扩容 | 手动升级 | 平台托管 |

---

## 部署步骤（从零开始）

### 1. 准备工作

```bash
# Node.js 项目，package.json 有 start 脚本
npm init
npm install pg docx
```

### 2. 创建 Supabase 数据库

1. 打开 [supabase.com](https://supabase.com) → Sign up (GitHub 登录)
2. 创建项目 → 记住密码
3. 进入 Settings → Database → Connection string → 复制 URI
4. 格式：`postgresql://postgres:[PASSWORD]@db.xxxxx.supabase.co:5432/postgres`

### 3. 创建 Render 服务

1. 打开 [render.com](https://render.com) → Sign up (GitHub 登录)
2. **New** → **Web Service** → 连接 GitHub 仓库
3. 配置：
   - **Build Command**: `npm install`
   - **Start Command**: `npm start`
   - **Plan**: Free
4. 添加环境变量：
   - `HOST` = `0.0.0.0`
   - `NODE_VERSION` = `20.x`
   - `DATABASE_URL` = Supabase 连接字符串（**机密，不要提交到 Git！**）

### 4. 防休眠（二选一）

**方案 A：GitHub Actions（推荐，代码即配置）**

本仓库已包含 `.github/workflows/keep-alive.yml`，push 到 GitHub 即生效，每 14 分钟自动 ping。

**方案 B：外部服务**

- [cron-job.org](https://cron-job.org) — 免费，每 10 分钟 ping 你的 `/api/health`
- [UptimeRobot](https://uptimerobot.com) — 免费，每 5 分钟 ping + 宕机邮件告警

### 5. 自动部署（二选一）

**方案 A：Render Blueprint Auto Sync**

1. 打开 [Render Dashboard](https://dashboard.render.com) → Blueprint 页面
2. Settings → **Sync** → 改为 **Automatic**
3. 之后每次 `git push` 自动部署

**方案 B：GitHub Actions Deploy Hook（更灵活）**

1. Render Dashboard → 服务 → Settings → Deploy Hooks → 复制 URL
2. GitHub 仓库 → Settings → Secrets → Actions → 添加 `RENDER_DEPLOY_HOOK`
3. 本仓库的 `.github/workflows/deploy.yml` 会在每次 push 时触发部署

### 6. 验证

```bash
# 1. 健康检查
curl https://your-app.onrender.com/api/health
# → {"ok":true}

# 2. 看日志
# Render Dashboard → 服务 → Logs

# 3. 测试数据库
# Supabase Dashboard → SQL Editor → SELECT * FROM users;
```

---

## 文件说明

```
your-project/
├── index.html              # 前端页面
├── app.js                  # 前端逻辑
├── styles.css              # 样式
├── server.mjs              # Node.js 后端
├── package.json            # npm 配置
├── render.yaml             # Render Blueprint 配置
├── DEPLOY.md               # ← 你正在看的文件
├── .github/workflows/
│   ├── keep-alive.yml      # 防休眠心跳
│   └── deploy.yml          # 自动部署
└── server/
    └── data/               # 文件存储降级方案（PostgreSQL 不可用时）
```

---

## 成本明细

| 服务 | 免费额度 | 用量 | 费用 |
|------|---------|------|------|
| Render Web Service | 750 hr/mo, 512MB, 100GB | 1 实例 24/7 | **$0** |
| Supabase PostgreSQL | 500MB, 2 projects | 1 项目 | **$0** |
| GitHub | 无限 public repo + Actions 2000min/mo | ~500 min/mo | **$0** |
| **合计** | | | **$0/月** |

### 何时需要升级

| 触发条件 | 升级方案 | 费用 |
|---------|---------|------|
| 用户 > 100 或并发高 | Render Starter | $7/mo |
| 数据 > 500MB | Supabase Pro | $25/mo |
| 需要中国加速 | Cloudflare CDN + 自定义域名 | 域名 ~¥50/年 |
| 需要 99.9% SLA | Render + Supabase 付费 + 多区域 | ~$50/mo |

---

## 常见问题

**Q: 免费 tier 会丢数据吗？**
A: 不会。Supabase 是持久化 PostgreSQL，有自动备份。Render 免费 tier 的磁盘是临时的，但我们不用它存数据。

**Q: 冷启动多久？**
A: 首次访问约 30-60 秒。加了 GitHub Actions 心跳后，基本不会触发休眠。如果还是遇到了，刷新等半分钟即可。

**Q: 安全性如何？**
A: 密码 scrypt 哈希存储，API Key 服务端加密（base64），HTTPS 自动，Render 和 Supabase 都有 DDoS 防护。每个用户用自己 Key，互不影响。

**Q: 能商用吗？**
A: Render 免费 tier 和 Supabase 免费 tier 都允许商用。但如果业务起来了，建议升级付费方案获得更好性能和 SLA。

**Q: 国内访问慢怎么办？**
A: Render 部署在 Oregon（美国西部），国内访问延迟 ~150-200ms。可加 Cloudflare CDN 加速静态资源。API 延迟本身受限于 AI 模型的响应速度，额外网络延迟影响不大。

---

## 相关链接

- [Render Dashboard](https://dashboard.render.com)
- [Supabase Dashboard](https://supabase.com/dashboard)
- [GitHub](https://github.com) — 托管代码 + Actions
