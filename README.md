# Claude2API — 部署指南

Claude.ai Web → Anthropic API 兼容代理。将 Claude.ai 账号会话转换为标准 `/v1/messages` 接口，支持流式输出、工具调用、图片上传、扩展思考等功能。

---

## 快速部署

### 1. 克隆此仓库

```bash
git clone https://github.com/realnoob007/claude2api-deploy.git
cd claude2api-deploy
```

### 2. 配置环境变量

```bash
cp .env.example .env
nano .env   # 填入你的配置
```

### 3. 启动服务

```bash
docker compose up -d
```

### 4. 验证

```bash
curl http://localhost:8080/health
# {"status":"ok","accounts":1}
```

---

## 两种运行模式

### 模式一：Simple 模式（推荐新手，无需数据库）

在 `.env` 中设置：

```env
CLAUDE_SESSION_KEYS=sk-ant-sid01-xxx,sk-ant-sid01-yyy
```

多个 Session Key 用逗号分隔，服务启动后自动轮询使用，无需 PostgreSQL。

### 模式二：PostgreSQL 模式（推荐生产）

`docker-compose.yml` 已内置 PostgreSQL 和 Redis，**开箱即用，无需额外安装**。

在 `.env` 中配置（`DB_HOST` 固定填 `postgres`，对应 compose 内的服务名）：

```env
DB_HOST=postgres
DB_PORT=5432
DB_USER=claude2api
DB_PASS=your_strong_password    # 必填，自定义强密码
DB_NAME=claude2api

REDIS_PASS=your_redis_password  # 必填，需与 REDIS_URL 保持一致
REDIS_URL=redis://:your_redis_password@redis:6379/0
```

> 如需连接**外部**数据库，将 `DB_HOST` 改为远程 IP/域名，并在 `docker-compose.yml` 中删除 `postgres` 和 `redis` 服务及对应的 `depends_on`。

通过管理面板动态添加/删除账号，支持每账号独立限额、封禁自动切换等高级功能。

---

## 获取 Session Key

1. 打开 [claude.ai](https://claude.ai) 并登录
2. 打开浏览器开发者工具（F12）→ Application → Cookies
3. 找到 `sessionKey`，复制其值（格式：`sk-ant-sid01-...`）

---

## 配置说明

| 变量 | 默认值 | 说明 |
|---|---|---|
| `LISTEN_ADDR` | `:8080` | 服务监听地址，格式为 `:端口号` |
| `CLAUDE_SESSION_KEYS` | — | Simple 模式：逗号分隔的 Session Keys |
| `DB_HOST` | `localhost` | PostgreSQL 主机 IP 或域名 |
| `DB_PORT` | `5432` | PostgreSQL 端口（默认 5432，远程可能不同）|
| `DB_USER` | `claude2api` | PostgreSQL 用户名 |
| `DB_PASS` | — | PostgreSQL 密码 |
| `DB_NAME` | `claude2api` | PostgreSQL 数据库名 |
| `PROXY_HOST` | — | 住宅代理主机（留空则直连 claude.ai）|
| `PROXY_PORT` | `3010` | 代理端口 |
| `PROXY_USER` | — | 代理认证用户名 |
| `PROXY_PASS` | — | 代理认证密码 |
| `PROXY_REGION` | `US` | 代理出口区域（如 `US`、`JP`、`GB`）|
| `ADMIN_USER` | `admin` | 管理面板用户名 |
| `ADMIN_PASS` | `admin` | 管理面板密码（**务必修改**）|
| `REDIS_URL` | — | Redis 连接串，用于持久化 metrics（可选）|
| `CLAUDE_DAILY_LIMIT` | `0` | 每账号每日请求上限（`0` = 不限）|
| `MAX_RETRIES` | `3` | 账号切换重试次数 |
| `COOLDOWN_MINUTES` | `5` | 触发 429 后冷却时间（分钟）|
| `CLAUDE_API_KEY` | — | 保护 `/v1/messages` 的 API Key（可选）|

---

## 数据库连接说明

默认使用 `docker-compose.yml` 内置的 PostgreSQL 容器，`DB_HOST` 固定填服务名 `postgres`，无需额外安装。

```env
# 默认：使用内置 postgres 容器
DB_HOST=postgres
DB_PORT=5432
DB_USER=claude2api
DB_PASS=your_strong_password
DB_NAME=claude2api
```

如需连接外部 PostgreSQL（托管云数据库等），将 `DB_HOST` 改为对应 IP 或域名，`DB_PORT` 改为实际端口，并删除 `docker-compose.yml` 中的 `postgres` 服务和 `depends_on` 相关配置。

```env
# 示例：外部托管数据库
DB_HOST=db.example.com
DB_PORT=5433
DB_USER=claude2api
DB_PASS=your_strong_password
DB_NAME=claude2api
```

---

## Redis 连接说明

默认使用 `docker-compose.yml` 内置的 Redis 容器。`REDIS_PASS` 和 `REDIS_URL` 中的密码必须保持一致。

```env
# 默认：使用内置 redis 容器
REDIS_PASS=your_redis_password
REDIS_URL=redis://:your_redis_password@redis:6379/0
```

如需连接外部 Redis，将 `REDIS_URL` 中的主机名改为实际地址，并删除 `docker-compose.yml` 中的 `redis` 服务和 `depends_on` 相关配置：

```env
# 示例：外部 Redis（带密码，非标准端口）
REDIS_URL=redis://:your_redis_password@redis.example.com:6388/0
```

连接串通用格式：
```
redis://:密码@主机:端口/库编号
```

---

## 代理配置说明

`PROXY_HOST`、`PROXY_PORT`、`PROXY_USER`、`PROXY_PASS`、`PROXY_REGION` 共同组成代理连接信息。留空则直连 claude.ai。

代理地址由服务内部按以下规则自动拼接：

| 场景 | 实际效果 |
|---|---|
| 仅填 `PROXY_HOST` / `PROXY_PORT` | 直连代理，无认证 |
| 填写 `PROXY_USER` / `PROXY_PASS` | 固定用户名密码认证 |
| 用户名含 `{sid}` 占位符 | 按账号固定出口 IP（旋转住宅代理）|
| 用户名含 `region-XX` | 指定出口区域（arxlabs 等服务商格式）|

管理面板中也可为每个账号单独配置代理，支持以下格式：

```
# 直连（不走代理）
留空

# 固定代理（无认证）
http://proxy.example.com:3128

# 固定代理（用户名密码认证）
http://user:pass@proxy.example.com:3010

# 旋转住宅代理（按账号固定 IP，{sid} 会被替换为账号唯一标识）
http://user-session-{sid}:pass@us.arxlabs.io:3010

# 旋转住宅代理 + 指定区域（arxlabs / brightdata 等常见格式）
http://user-region-US-session-{sid}:pass@us.arxlabs.io:3010
http://user-country-JP-session-{sid}:pass@proxy.provider.io:3010
```

---

## API 使用

服务兼容 Anthropic Messages API：

```bash
curl http://localhost:8080/v1/messages \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $CLAUDE_API_KEY" \
  -d '{
    "model": "claude-sonnet-4-6",
    "max_tokens": 1024,
    "messages": [{"role": "user", "content": "你好！"}]
  }'
```

未设置 `CLAUDE_API_KEY` 时，`Authorization` 头可省略。

### 支持的模型名

| 请求模型名（示例） | 实际路由 |
|---|---|
| `claude-sonnet-4-6` | claude-sonnet-4-6 |
| `claude-sonnet-4-5` | claude-sonnet-4-5-20250929 |
| `claude-haiku-4-5` | claude-haiku-4-5-20251001 |
| `claude-opus-4-6` | claude-opus-4-6（需要 Pro）|
| `*-thinking` 后缀 | 同模型 + 开启扩展思考 |
| 自定义映射 | 在管理面板配置 |

---

## 管理面板

访问 `http://localhost:8080/admin`，使用 `ADMIN_USER` / `ADMIN_PASS` 登录。

功能：
- 账号管理（添加 / 批量导入 / 删除 / 状态监控）
- 代理配置（支持 `{sid}` 占位符实现按账号固定 IP）
- 模型名映射（自定义入参模型名 → Claude.ai 模型名）
- 实时 metrics（请求数、延迟、成功率）
- 全局限速配置
- API Key 管理
- 支持中文 / English 切换，深色 / 浅色主题

---

## 更新镜像

```bash
docker compose pull
docker compose up -d
```
