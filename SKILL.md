---
name: free-web-deploy
description: 免费网页测试部署 — 把带后端+数据库的网站用 Render + Supabase + GitHub Actions 零成本上线。适合原型验证、学习演示、小团队内部工具。Use when the user wants to deploy a full-stack web app for free, set up a database-backed website at zero cost, or needs a step-by-step guide to go from code to live URL without paying for a server.
---

# 🆓 免费网页部署 Skill：不花钱，把你的网站放上互联网

> 🤖 AI 生成的部署流程。对着做，每一步做完都有 ✅ 检查点。卡住了把这段话发给 AI。

## 这个 Skill 能干什么

把你的项目从"本地能跑"变成"别人能访问的公网网址"，全程 $0。

```
你的电脑 → GitHub（存代码）→ Render（运行）→ 别人打开网址就能用
                        → Supabase（存数据）
                        → GitHub Actions（自动更新 + 防休眠）
```

---

## 开始之前（3 个账号）

坐下来先把这三个注册好，5 分钟：

| 账号 | 去哪注册 | 要什么 |
|------|---------|--------|
| GitHub | https://github.com → Sign up | 邮箱 + 用户名 |
| Render | https://render.com → Sign up → **点 GitHub 图标登录** | 授权 GitHub |
| Supabase | https://supabase.com → Sign up → **点 GitHub 图标登录** | 授权 GitHub |

> ✅ 检查点：三个网站都登录着，不用关标签页。

---

## Step 1：准备你的项目

确保项目根目录有这两个文件：

**`package.json`** — 告诉 Render "这是 Node.js 项目，怎么启动"：

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

**`server.mjs`**（或用你自己的文件名）——必须包含一个健康检查端点：

```js
import http from "http";

const PORT = process.env.PORT || 8787;
const HOST = process.env.HOST || "127.0.0.1";

// ===== 数据库连接 =====
import pg from "pg";
const pool = process.env.DATABASE_URL
  ? new pg.Pool({
      connectionString: process.env.DATABASE_URL,
      ssl: { rejectUnauthorized: false }
    })
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
    email TEXT NOT NULL, created_at BIGINT NOT NULL,
    expires_at BIGINT NOT NULL)`);
  await dbQuery(`CREATE TABLE IF NOT EXISTS user_settings (
    user_id TEXT PRIMARY KEY, data JSONB DEFAULT '{}')`);
  await dbQuery(`CREATE TABLE IF NOT EXISTS user_conversations (
    user_id TEXT PRIMARY KEY, data JSONB DEFAULT '{}')`);
}

// ===== HTTP 服务 =====
const server = http.createServer(async (req, res) => {
  res.setHeader("Access-Control-Allow-Origin", "*");
  if (req.method === "OPTIONS") { res.writeHead(204); res.end(); return; }

  // ✅ 必须有这个端点！Render 用它判断服务是否存活
  if (req.method === "GET" && req.url === "/api/health") {
    res.writeHead(200, { "Content-Type": "application/json" });
    return res.end(JSON.stringify({ ok: true }));
  }

  // 你的业务路由写在这里...
});

initDB().then(() => {
  server.listen(PORT, HOST, () => console.log(`http://${HOST}:${PORT}`));
});
```

> ✅ 检查点：`npm install && npm start` 能在本地跑起来。

---

## Step 2：创建 Supabase 数据库

1. 打开 https://supabase.com/dashboard → **New project**
2. 填项目名（随便，比如 `my-app-db`）
3. 设数据库密码 → **记下来，后面要用**
4. Region 选离你近的（亚洲选 Singapore 或 Tokyo）
5. 点 **Create new project** → 等 1 分钟初始化
6. 进 **Settings → Database → Connection string** → 选 **URI** → 复制

你会拿到一串这样的：
```
postgresql://postgres:[你的密码]@db.xxxxx.supabase.co:5432/postgres
```

> ✅ 检查点：已复制连接字符串，密码没忘。

---

## Step 3：创建 render.yaml

项目根目录新建 `render.yaml`，内容：

```yaml
services:
  - type: web
    name: 你的项目名        # ← 改这里，英文小写+连字符，如 my-test-app
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

> ⚠️ `HOST: 0.0.0.0` 不能改成别的，否则 Render 访问不到你的服务。

> ✅ 检查点：文件已创建，`name` 已改。

---

## Step 4：添加 GitHub Actions

创建两个文件：

**`.github/workflows/keep-alive.yml`** — 每 14 分钟 ping 一次，防止 Render 休眠：

```yaml
name: Keep Render Alive
on:
  schedule:
    - cron: '*/14 * * * *'
  workflow_dispatch:

jobs:
  ping:
    runs-on: ubuntu-latest
    steps:
      - name: Ping health check
        run: |
          STATUS=$(curl -s -o /dev/null -w "%{http_code}" https://改成你的项目名.onrender.com/api/health)
          echo "Status: $STATUS"
          if [ "$STATUS" != "200" ]; then
            echo "::warning::Health check returned $STATUS"
          fi
```

> ⚠️ 把 `改成你的项目名` 换成 Step 3 里 `render.yaml` 的 `name`。

**`.github/workflows/deploy.yml`** — git push 自动部署：

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

> ✅ 检查点：`.github/workflows/` 下有两个 `.yml` 文件。

---

## Step 5：推到 GitHub 并首次部署

```bash
git add .
git commit -m "初次部署"
git push origin main
```

然后去 Render 部署：

1. 打开 https://dashboard.render.com → **New** → **Blueprint**
2. 连接你的 GitHub 仓库 → Render 自动识别 `render.yaml`
3. 点 **Apply** → 等 3-5 分钟

> ✅ 检查点：Render Dashboard 里服务状态是 **Live**（绿色）。

---

## Step 6：绑定数据库（关键一步）

部署成功后，把 Supabase 数据库连上：

1. Render Dashboard → 点你的服务 → **Environment**
2. 添加一行：
   | Key | Value |
   |-----|-------|
   | `DATABASE_URL` | Step 2 复制的那串 `postgresql://...` |
3. 点 **Save** → Render 会自动重启服务

> ⚠️ **绝对不要把 `DATABASE_URL` 写进代码或 render.yaml！** 密码会在 GitHub 上被所有人看到。
> 
> ✅ 检查点：服务重启后，打开 `https://你的项目名.onrender.com/api/health` 看到 `{"ok":true}`。

---

## Step 7：开启自动部署

1. Render Dashboard → 点你的 Blueprint → **Settings**
2. **Sync** → 改为 **Automatic**
3. 之后每次 `git push`，Render 自动更新

如果不行（比如没 Blueprint），手动配 Deploy Hook：

1. Render Dashboard → 服务 → **Settings** → **Deploy Hooks** → 复制 URL
2. GitHub 仓库 → **Settings** → **Secrets → Actions** → 添加 `RENDER_DEPLOY_HOOK`，粘贴刚才的 URL

> ✅ 检查点：改一行代码 → git push → Render 自动开始部署。

---

## 最终验证清单

- [ ] 浏览器打开 `https://你的项目名.onrender.com` 能访问
- [ ] `https://你的项目名.onrender.com/api/health` 返回 `{"ok":true}`
- [ ] 等 15 分钟不操作 → 再次打开仍然秒开（心跳生效）
- [ ] Supabase Dashboard → SQL Editor → `SELECT * FROM users;` 有数据

---

## ⚠️ 免费限制

| 服务 | 免费额度 | 坑 |
|------|---------|-----|
| Render | 750 小时/月，512MB 内存 | 15 分钟没人访问就休眠（本方案用心跳解决）；磁盘是临时的，重启就清空 |
| Supabase | 500MB 数据库 | 90 天不活跃项目会被暂停（会邮件提醒） |
| GitHub Actions | 2000 分钟/月 | 仓库 60 天无提交，定时任务自动停 |

---

## 常见故障

| 症状 | 原因 | 解决 |
|------|------|------|
| 部署完打不开，404 | `HOST` 没设 `0.0.0.0` | 检查 render.yaml envVars |
| 打开要等 30-60 秒 | Render 休眠了 | 等 14 分钟让心跳生效 |
| 存的数据重启就没了 | 没配 `DATABASE_URL`，数据写到 Render 磁盘了 | 去 Render Environment 加上 `DATABASE_URL` |
| `pg` 连不上 | Supabase 默认 SSL 连接 | 代码里 `ssl: { rejectUnauthorized: false }` |
| `git push` 但不自动部署 | 没用 SSH remote 或 Blueprint 没开 Auto Sync | `git remote set-url origin git@github.com:...` |

---

## 如何安装这个 Skill

### 方法 1：直接复制文件到你项目

```bash
# 在你的项目根目录
cp render.yaml .github/workflows/keep-alive.yml .github/workflows/deploy.yml 你的项目路径/
```

然后改 `render.yaml` 的 `name` 和 `keep-alive.yml` 的 URL。

### 方法 2：让 AI 按这个流程帮你

把 https://github.com/rmxhah/free-fullstack-deploy 发给 Claude，说"按这个 SKILL 帮我部署"。

---

## 相关链接

- 本模板仓库：https://github.com/rmxhah/free-fullstack-deploy
- Render：https://render.com
- Supabase：https://supabase.com
- GitHub Actions：https://docs.github.com/en/actions
