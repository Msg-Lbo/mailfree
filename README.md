# Mailfree 临时邮箱（Cloudflare Workers）

基于 Cloudflare Workers + D1 + R2 的临时邮箱系统，支持收件、发件、用户权限和管理后台。

## 功能概览

- 临时邮箱生成：支持邮箱前缀 + 随机字符串长度规则
- 域名动态管理：在管理后台实时添加/删除域名（不再依赖 `MAIL_DOMAIN`）
- 历史邮箱与置顶：支持分页、置顶、删除
- 邮件收件箱：列表、详情、批量查询、删除、清空、EML 下载
- 发件箱（Resend）：单发、批量、查询状态、取消、删除记录
- 用户体系：严格管理员 / 高级用户 / 普通用户 / 访客（guest）
- 账户数据清理：支持一键清空某个账户下全部邮箱及相关数据

## 技术栈

- Runtime: Cloudflare Workers (Modules)
- Database: Cloudflare D1
- Object Storage: Cloudflare R2（存储原始 EML）
- Frontend: 原生 HTML/CSS/JS（`public/`）

## 快速开始（已按当前代码校对）

### 1) 前置要求

- Node.js 18+
- npm
- Cloudflare 账号（已开通 Workers / D1 / R2 / Email Routing）
- Wrangler CLI

安装 Wrangler 并登录：

```bash
npm i -g wrangler
wrangler login
```

### 2) 克隆项目

```bash
git clone <你的仓库地址>
cd mailfree
```

### 3) 创建 D1 与 R2

```bash
# 创建 D1
wrangler d1 create mailfree

# 创建 R2（用于邮件原文 .eml）
wrangler r2 bucket create mailfree
```

### 4) 配置 `wrangler.toml`

请将 `wrangler.toml` 中绑定改成你自己的资源：

```toml
name = "mailfree"
main = "src/server.js"
compatibility_date = "2024-01-01"

[[d1_databases]]
binding = "TEMP_MAIL_DB"
database_name = "你的D1名称"
database_id = "你的D1 ID"

[[r2_buckets]]
binding = "MAIL_EML"
bucket_name = "你的R2桶名"

[assets]
directory = "public"
binding = "ASSETS"

[vars]
ADMIN_NAME = "admin"
```

### 5) 配置 Secrets / Variables

必需（建议用 Secret）：

```bash
wrangler secret put ADMIN_PASSWORD
wrangler secret put JWT_TOKEN
```

可选：

```bash
# 访客模式（登录用户名固定 guest）
wrangler secret put GUEST_PASSWORD

# 发件功能（Resend）
wrangler secret put RESEND_API_KEY

# 邮件转发规则（可选，也可放 Variables）
wrangler secret put FORWARD_RULES
```

### 6) 初始化数据库表结构

```bash
wrangler d1 execute <你的D1名称> --file=./d1-init-basic.sql --remote
```

本地调试库初始化（可选）：

```bash
wrangler d1 execute <你的D1名称> --file=./d1-init-basic.sql --local
```

### 7) 本地运行

```bash
wrangler dev
```

打开：`http://127.0.0.1:8787/html/login.html`

## 首次上线后的必做步骤

现在域名为**全动态配置**，没有默认域名。

1. 用严格管理员登录（`ADMIN_NAME` / `ADMIN_PASSWORD`）
2. 进入 `/html/admin.html`
3. 在「域名管理」里添加至少 1 个域名

未添加域名前，`/api/generate` 与 `/api/create` 会返回 400（无法生成邮箱）。

## 部署到 Cloudflare

```bash
wrangler deploy
```

如果你使用 Cloudflare Dashboard 的 Git 集成部署，请在 Worker 设置中同步配置：

- Secrets：`ADMIN_PASSWORD`、`JWT_TOKEN`（以及可选项）
- Variables：`ADMIN_NAME`（可选）、`FORWARD_RULES`（可选）
- Bindings：D1 / R2 / Assets

## 邮件接收配置（非常重要）

每个你在后台添加的域名，都要在 Cloudflare Email Routing 完成配置：

1. 域名 DNS/MX 配置正确
2. Email Routing 开启并生效
3. Catch-all 或对应规则指向当前 Worker

否则该域名虽然可用于“生成地址”，但收不到真实邮件。

## 环境变量与绑定说明

| 名称 | 类型 | 必需 | 说明 |
|---|---|---|---|
| `TEMP_MAIL_DB` | D1 绑定 | 是 | 主数据库 |
| `MAIL_EML` | R2 绑定 | 是 | 邮件原文存储（下载/详情依赖） |
| `ASSETS` | Assets 绑定 | 是 | 托管 `public/` 静态资源 |
| `ADMIN_PASSWORD` | Secret | 是 | 严格管理员密码 |
| `JWT_TOKEN` | Secret | 是 | JWT 签名密钥 |
| `ADMIN_NAME` | Variable | 否 | 严格管理员用户名，默认 `admin` |
| `GUEST_PASSWORD` | Secret | 否 | 启用访客账号 `guest` |
| `RESEND_API_KEY` | Secret | 否 | 启用发件功能 |
| `FORWARD_RULES` | Variable/Secret | 否 | 本地前缀转发规则 |

兼容别名（不推荐，仅兼容旧配置）：

- `JWT_SECRET`（等价 `JWT_TOKEN`）
- `ADMIN_PASS`（等价 `ADMIN_PASSWORD`）
- `RESEND_TOKEN` / `RESEND`（等价 `RESEND_API_KEY`）

## FORWARD_RULES 示例

- KV 形式：
  - `vip=a@example.com,news=b@example.com,*=fallback@example.com`
- JSON 数组：
  - `[{"prefix":"vip","email":"a@example.com"},{"prefix":"*","email":"fallback@example.com"}]`
- 关闭转发：
  - 空字符串、`disabled`、`none`、`[]`

## API 文档

- 完整接口文档：`docs/api.md`
- Resend 发件说明：`docs/resend.md`

## 常见问题

### 1) 登录失败

- 检查 `ADMIN_PASSWORD`、`JWT_TOKEN` 是否已配置
- 严格管理员用户名默认是 `admin`，可通过 `ADMIN_NAME` 改

### 2) 生成邮箱提示无可用域名

- 进入管理后台添加域名
- 确认该域名符合格式（如 `mail.example.com`）

### 3) 域名能生成地址但收不到邮件

- 检查该域名的 Email Routing / MX / Catch-all 是否已绑定 Worker

### 4) 邮件详情为空或无法下载

- 检查 `MAIL_EML`（R2）绑定是否正确

## 许可证

Apache-2.0
