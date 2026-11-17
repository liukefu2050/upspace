# UpSpace 本地部署指南

> 本指南优先使用本地服务，外部服务仅列出注册地址与步骤。

---

## 前置工具

| 工具 | 要求 | 安装方式 |
|---|---|---|
| Node.js | 最新版即可（当前 v24+） | https://nodejs.org |
| pnpm | 最新版 | `npm install -g pnpm` |
| Docker Desktop | 必装（本地 Supabase/Redis 依赖） | https://www.docker.com/products/docker-desktop |
| Git | 任意版本 | https://git-scm.com |

确认：

```bash
node --version
pnpm --version
docker --version
```

---

## 第一步：启动本地服务

### 1.1 本地 Supabase（数据库 + Auth + Storage + 邮件测试）

项目已含 [supabase/config.toml](supabase/config.toml)，无需额外配置。

```bash
pnpm dlx supabase start
```

启动后运行以下命令获取本地凭据：

```bash
pnpm dlx supabase status
```

记录输出中的：

| 项 | 示例值 | 对应 .env 变量 |
|---|---|---|
| API URL | `http://127.0.0.1:54321` | `NEXT_PUBLIC_SUPABASE_URL` |
| anon key | `eyJ...`（本地生成的 JWT） | `NEXT_PUBLIC_SUPABASE_ANON_KEY` |
| service_role key | `eyJ...`（本地生成的 JWT） | `SUPABASE_SERVICE_ROLE_KEY` |
| DB URL | `postgresql://postgres:postgres@127.0.0.1:54322/postgres` | `DATABASE_URL` |

本地服务端口：

| 服务 | 端口 | 用途 |
|---|---|---|
| Supabase API | 54321 | Auth + Storage API |
| PostgreSQL | 54322 | 数据库 |
| Supabase Studio | 54323 | 可视化管理 http://127.0.0.1:54323 |
| Inbucket | 54324 | 邮件测试界面 http://127.0.0.1:54324 |

> 本地 Supabase 已内置 PostGIS、pg_trgm、uuid-ossp 扩展，无需手动启用。
> Inbucket 会拦截所有邮件，在 http://127.0.0.1:54324 可查看 OTP 验证码。

### 1.2 本地 Redis（缓存 + 限流）

```bash
docker run -d --name upspace-redis -p 6379:6379 redis:7-alpine
```

对应 `.env`：

```
REDIS_URL="redis://localhost:6379"
```

---

## 第二步：注册外部服务（按需）

以下服务不注册则对应功能不可用，核心 marketplace 流程不受影响。

### 2.1 Xendit（支付/退款/提现）

注册地址：https://dashboard.xendit.co/register

步骤：
1. 选择 **Philippines** 区域注册
2. 进入 Dashboard → Settings → API Keys
3. 复制 **Secret Key**（以 `xnd_development_` 开头的为测试密钥）
4. 进入 Settings → Webhooks，添加回调地址 `https://<你的隧道域名>/api/provider/webhook`
5. 复制 Webhook Verification Token

对应 `.env`：

```
XENDIT_SECRET_KEY="xnd_development_xxxxx"
XENDIT_WEBHOOK_VERIFICATION_TOKEN="xxxxx"
FINANCIAL_DATA_ENCRYPTION_KEY="替换为一串随机长字符串"
```

> 本地测试 webhook 需用隧道工具（如 ngrok）暴露 3000 端口：
> `ngrok http 3000` 然后把生成的 https 地址填入 Xendit Webhook 配置。

### 2.2 OpenRouter（AI 助手）

注册地址：https://openrouter.ai/

步骤：
1. 注册账号
2. 进入 https://openrouter.ai/keys 创建 API Key
3. 复制 Key

对应 `.env`：

```
OPENROUTER_API_KEY="sk-or-v1-xxxxx"
OPENROUTER_MODEL="anthropic/claude-haiku-4.5"
```

### 2.3 Google Maps（地址自动补全，可选）

注册地址：https://console.cloud.google.com/

步骤：
1. 创建新项目
2. 启用 **Maps JavaScript API** 和 **Places API**
3. 进入 APIs & Services → Credentials → 创建 API Key
4. 限制 Key 的引用来源为 `http://localhost:3000`

对应 `.env`：

```
NEXT_PUBLIC_GOOGLE_MAPS_API_KEY="AIzaxxxxx"
```

---

## 第三步：配置 .env

在项目根目录创建 `.env`，填入已收集的值：

```bash
# ===== 核心（必填）=====
DATABASE_URL="postgresql://postgres:postgres@127.0.0.1:54322/postgres"
NEXT_PUBLIC_SUPABASE_URL="http://127.0.0.1:54321"
NEXT_PUBLIC_SUPABASE_ANON_KEY="从 supabase status 获取"
SUPABASE_SERVICE_ROLE_KEY="从 supabase status 获取"
NEXT_PUBLIC_APP_URL="http://localhost:3000"
NEXT_PUBLIC_APP_NAME="UpSpace"

# ===== Storage =====
NEXT_PUBLIC_SPACE_IMAGES_BUCKET="space-images"
NEXT_PUBLIC_VERIFICATION_DOCS_BUCKET="verification-docs"

# ===== Redis（本地）=====
REDIS_URL="redis://localhost:6379"

# ===== AI 助手（注册后填入）=====
OPENROUTER_API_KEY="sk-or-v1-xxxxx"
OPENROUTER_MODEL="anthropic/claude-haiku-4.5"

# ===== 支付（注册后填入）=====
XENDIT_SECRET_KEY="xnd_development_xxxxx"
XENDIT_WEBHOOK_VERIFICATION_TOKEN="xxxxx"
FINANCIAL_DATA_ENCRYPTION_KEY="生成一串随机字符串"

# ===== 邮件（本地用 Inbucket，无需真实 SMTP）=====
EMAIL_SMTP_HOST="127.0.0.1"
EMAIL_SMTP_PORT="54325"
EMAIL_SMTP_SECURE="false"
EMAIL_SMTP_USER=""
EMAIL_SMTP_PASSWORD=""
EMAIL_FROM="noreply@upspace.local"
EMAIL_FROM_NAME="UpSpace"
```

> Inbucket SMTP 端口为 54325（见 [supabase/config.toml](supabase/config.toml#L98)）。
> 如不测试邮件功能，SMTP 相关变量可留空，OTP 会在 Inbucket 界面可见。

---

## 第四步：安装与启动

```bash
# 安装依赖（自动执行 prisma generate）
pnpm install

# 应用数据库迁移
pnpm prisma migrate dev

# 生成 API 文档（可选）
pnpm docs:api

# 启动开发服务器
pnpm dev
```

---

## 第五步：验证

```bash
pnpm lint
pnpm test
pnpm build
```

访问地址：

| URL | 用途 |
|---|---|
| http://localhost:3000 | 产品 UI |
| http://localhost:3000/docs | API 文档 |
| http://127.0.0.1:54323 | Supabase Studio |
| http://127.0.0.1:54324 | Inbucket 邮件查看 |

---

## 常见问题

| 问题 | 解决方案 |
|---|---|
| `supabase start` 失败 | 确认 Docker Desktop 已启动且正常运行 |
| Prisma 迁移报扩展错误 | 本地 Supabase 已内置扩展；如用自建 PG 需手动执行 `create extension postgis; create extension pg_trgm; create extension "uuid-ossp";` |
| 登录后 profile 路由失败 | 检查 `SUPABASE_SERVICE_ROLE_KEY` 是否正确填写 |
| AI 助手报配置错误 | 检查 `OPENROUTER_API_KEY` 是否有效 |
| 钱包/支付功能失败 | 检查 `XENDIT_SECRET_KEY` 和 `FINANCIAL_DATA_ENCRYPTION_KEY` |
| 收不到 OTP 邮件 | 本地模式查看 http://127.0.0.1:54324 Inbucket 界面 |
| 端口冲突 | 修改 [supabase/config.toml](supabase/config.toml) 中对应端口 |

---

## 停止服务

```bash
# 停止本地 Supabase
pnpm dlx supabase stop

# 停止 Redis
docker stop upspace-redis
```
