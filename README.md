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

在 `.env` 中配置数据库连接：

```env
DB_HOST=your-postgres-host
DB_PORT=5432
DB_USER=claude2api
DB_PASS=your_password
DB_NAME=claude2api
```

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
| `CLAUDE_SESSION_KEYS` | — | Simple 模式：逗号分隔的 Session Keys |
| `DB_HOST` | `localhost` | PostgreSQL 主机 |
| `DB_PORT` | `5432` | PostgreSQL 端口 |
| `DB_USER` | `claude2api` | PostgreSQL 用户名 |
| `DB_PASS` | — | PostgreSQL 密码 |
| `DB_NAME` | `claude2api` | PostgreSQL 数据库名 |
| `PROXY_HOST` | — | 住宅代理地址（留空直连） |
| `PROXY_PORT` | `3010` | 代理端口 |
| `PROXY_USER` | — | 代理用户名 |
| `PROXY_PASS` | — | 代理密码 |
| `PROXY_REGION` | `US` | 代理区域 |
| `ADMIN_USER` | `admin` | 管理面板用户名 |
| `ADMIN_PASS` | `admin` | 管理面板密码（务必修改）|
| `REDIS_URL` | — | Redis 地址，用于持久化 metrics（可选）|
| `CLAUDE_DAILY_LIMIT` | `0` | 每账号每日请求上限（0=不限）|
| `MAX_RETRIES` | `3` | 账号切换重试次数 |
| `COOLDOWN_MINUTES` | `5` | 触发 429 后冷却时间（分钟）|
| `CLAUDE_API_KEY` | — | 保护 `/v1/messages` 的 API Key（可选）|

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

## 代理格式说明

| 格式 | 说明 |
|---|---|
| 留空 | 直连 claude.ai |
| `http://user:pass@host:port` | 固定代理 |
| `http://user-session-{sid}:pass@host:port` | 旋转住宅代理，`{sid}` 按账号固定 IP |
| `http://user-region-US-session-{sid}:pass@host:port` | 区域路由（arxlabs 等格式）|

---

## 更新镜像

```bash
docker compose pull
docker compose up -d
```

---

## 源码

[github.com/realnoob007/claude2api](https://github.com/realnoob007/claude2api)
