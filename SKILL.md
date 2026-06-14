# Skill: 免费全栈部署 (Render + Supabase + GitHub)

## 元信息
- **调用名**: `free-fullstack-deploy`
- **用途**: 把带 Node.js 后端 + PostgreSQL 数据库的全栈应用，用完全免费的服务上线
- **月费**: $0
- **适用**: 原型验证、内部工具、学习项目、SaaS MVP
- **不适用**: 纯静态网站 → 直接用 GitHub Pages

---

## 它能做什么

给你一套"代码 push 到 GitHub → 自动上线 → 数据持久化 → 不休眠"的完整流程，替代以前"买 VPS + 配 Nginx + 配 HTTPS + 配数据库 + 写部署脚本"的繁琐操作。

---

## 前置条件

- [ ] GitHub 账号
- [ ] Render 账号（用 GitHub 登录，免费 tier）
- [ ] Supabase 账号（用 GitHub 登录，免费 tier）
- [ ] 项目有 `npm start` 脚本，服务监听 `process.env.PORT`

---

## Step 1：最简服务端模板

你的 `server.mjs`（或 `server.js`）需要包含以下关键部分：

```js
import http from "http";
import { env } from "process";

const PORT = env.PORT || 8787;
const HOST = env.HOST || "127.0.0.1";

// ===== 数据库（可选，Supabase PostgreSQL）=====
import pg from "pg";
const pool = env.DATABASE_URL
  ? new pg.Pool({ connectionString: env.DATABASE_URL, ssl: { rejectUnauthorized: false } })
  : null;

async function dbQuery(text, params) {
  if (!pool) throw new Error("No database configured");
  return pool.query(text, params);
}

async function initDB() {
  if (!pool) return;
  await dbQuery(`CREATE TABLE IF NOT EXISTS users (
    email TEXT PRIMARY KEY, user_id TEXT NOT NULL,
    password_hash TEXT NOT NULL, created_at BIGINT NOT NULL)`);
  await dbQuery(`CREATE TABLE IF NOT EXISTS sessions (
    token TEXT PRIMARY KEY, user_id TEXT NOT NULL,
    email TEXT NOT NULL, created_at BIGINT NOT NULL, expires_at BIGINT NOT NULL)`);
  await dbQuery(`CREATE TABLE IF NOT EXISTS user_settings (
    user_id TEXT PRIMARY KEY, data JSONB DEFAULT '{}')`);
  await dbQuery(`CREATE TABLE IF NOT EXISTS user_conversations (
    user_id TEXT PRIMARY KEY, data JSONB DEFAULT '{}')`);
}

// ===== HTTP 服务 =====
const server = http.createServer(async (req, res) => {
  // CORS
  res.setHeader("Access-Control-Allow-Origin", "*");
  if (req.method === "OPTIONS") return res.writeHead(204).end();

  // 健康检查（Render 需要 + 防休眠心跳用）
  if (req.method === "GET" && req.url === "/api/health") {
    res.writeHead(200, { "Content-Type": "application/json" });
    return res.end(JSON.stringify({ ok: true }));
  }

  // 你的业务路由...
});

// 启动时初始化数据库
initDB().then(() => {
  server.listen(PORT, HOST, () => console.log(`Listening on ${HOST}:${PORT}`));
});
```

**必装依赖**（`package.json`）:

```json
{
  "type": "module",
  "scripts": { "start": "node server.mjs" },
  "engines": { "node": ">=20" },
  "dependencies": {
    "pg": "^8.13.0"
  }
}
```

---

## Step 2：创建 render.yaml

```yaml
services:
  - type: web
    name: your-app-name          # ← 改这里
    env: node
    region: oregon
    buildCommand: npm install
    startCommand: npm start
    healthCheckPath: /api/health
    envVars:
      - key: HOST
        value: 0.0.0.0
      - key: NODE_VERSION
        value: 20.x
    plan: free
```

---

## Step 3：防休眠 — GitHub Actions

复制到 `.github/workflows/keep-alive.yml`:

```yaml
name: Keep Render Alive
on:
  schedule:
    # 每14分钟一次，避开整点/半点减少服务器拥挤
    - cron: '*/14 * * * *'
  workflow_dispatch:

jobs:
  ping:
    runs-on: ubuntu-latest
    steps:
      - name: Ping health check
        run: |
          STATUS=$(curl -s -o /dev/null -w "%{http_code}" https://YOUR-APP.onrender.com/api/health)
          echo "Status: $STATUS"
          if [ "$STATUS" != "200" ]; then
            echo "::warning::Health check returned $STATUS"
          fi
```

> ⚠️ GitHub 会在仓库 60 天无提交后停用定时任务。活跃项目不受影响。

---

## Step 4：自动部署

**方式 A — 最简单：Render Blueprint Auto Sync**

1. 打开 [Render Dashboard](https://dashboard.render.com) → 点你的 Blueprint
2. Settings → Sync → 改为 **Automatic**
3. 之后每次 `git push` 自动部署

**方式 B — 更灵活：Deploy Hook**

复制到 `.github/workflows/deploy.yml`:

```yaml
name: Deploy to Render
on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest
    if: ${{ secrets.RENDER_DEPLOY_HOOK != '' }}
    steps:
      - name: Trigger deploy
        run: curl -X POST "${{ secrets.RENDER_DEPLOY_HOOK }}"
```

然后在 GitHub 仓库 → Settings → Secrets → Actions → 添加 `RENDER_DEPLOY_HOOK`（值从 Render Dashboard → 服务 → Settings → Deploy Hooks 复制）。

---

## Step 5：首次部署

1. Push 代码到 GitHub
2. Render Dashboard → **New** → **Web Service** → 连接你的 GitHub 仓库
3. 或：**New** → **Blueprint** → 连接仓库（自动识别 `render.yaml`）
4. 添加环境变量 `DATABASE_URL` = `postgresql://postgres:[YOUR-PASSWORD]@db.xxxxx.supabase.co:5432/postgres`（⚠️ 在 Render Dashboard 手动加，**不要写进 render.yaml 或任何文件**）
5. 等 2-5 分钟部署完成

---

## Step 6：验证

```bash
# 健康检查
curl https://your-app.onrender.com/api/health
# → {"ok":true}

# 查看日志
# Render Dashboard → 服务 → Logs

# 验证数据库
# Supabase Dashboard → SQL Editor → SELECT * FROM users;
```

---

## 升级路径

| 阶段 | 方案 | 月费 |
|------|------|------|
| MVP / 个人项目 | 本方案 | **$0** |
| 有稳定用户 | Render Starter + Supabase Pro | ~$32/mo |
| 商业产品 | Render Standard + Supabase Team | ~$100+/mo |

---

## 常见坑

| 坑 | 原因 | 解决 |
|---|------|------|
| 部署后访问 404 | 服务监听 127.0.0.1 而非 0.0.0.0 | `HOST` 环境变量设为 `0.0.0.0` |
| 首次访问等 30-60s | 免费 tier 休眠 | 等 GitHub Actions 心跳生效（14 分钟内） |
| 数据隔天就没了 | 用了 Render 本地磁盘而非 Supabase | 确保 `DATABASE_URL` 已配，代码优先走 pg |
| `pg` 连接报错 | SSL 未配置 | `ssl: { rejectUnauthorized: false }` |
| GitHub Actions 不触发 | 60 天无提交被停用 | Push 任意 commit 重新激活 |
| Supabase 连不上 | IP 未加白名单 | Supabase Settings → 关闭 IP 限制或加 Render IP |

---

## 复制到新项目

```bash
# 复制核心部署文件
cp render.yaml .github/workflows/keep-alive.yml .github/workflows/deploy.yml 目标项目/

# 修改点：
# 1. render.yaml 里的 name
# 2. keep-alive.yml 里的 YOUR-APP.onrender.com
# 3. 确保 server 有 /api/health 端点
# 4. 确保 server 用 DATABASE_URL 连接数据库
```

---

## 相关链接

- Render: https://render.com
- Supabase: https://supabase.com
- GitHub Actions 文档: https://docs.github.com/en/actions
