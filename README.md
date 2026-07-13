# Sub2API Grok CLI 增强版

> 基于 [Wei-Shaw/sub2api](https://github.com/Wei-Shaw/sub2api) 的完整可用源码仓库  
> **克隆即可构建部署**，额外增强了 **Grok CLI / CLIProxyAPI 账号兼容**

[![Go](https://img.shields.io/badge/Go-1.25+-00ADD8.svg)](https://golang.org/)
[![Vue](https://img.shields.io/badge/Vue-3.4+-4FC08D.svg)](https://vuejs.org/)
[![Docker](https://img.shields.io/badge/Docker-Ready-2496ED.svg)](https://www.docker.com/)
[![Upstream](https://img.shields.io/badge/upstream-Wei--Shaw%2Fsub2api-blue.svg)](https://github.com/Wei-Shaw/sub2api)

---

## 这是什么？

**Sub2API** 是一个 AI API 网关平台，可以把多平台订阅账号（Claude / OpenAI / Gemini / Grok 等）汇聚成统一的 OpenAI 兼容接口，做配额分发、账号调度、故障切换。

本仓库是 **完整源码 fork**，在上游基础上专门修好了 **Grok CLI OAuth 账号** 的兼容问题，方便直接导入 CLIProxyAPI / Grok Shell 产出的 auth 文件并稳定跑 `grok-4.5` 等模型。

别人克隆这个仓库后，可以：

1. 用 Docker Compose 一键部署完整 Sub2API
2. 导入 Grok CLI 账号（含 `headers`、`expired` 字段的 auth 文件）
3. 通过 `/v1/chat/completions` 调用 Grok 模型

---

## 相对上游新增了什么？

| 改动 | 作用 |
|---|---|
| 兼容 `model` / `model_id` | 后台「测试账号」接口两种字段都能用 |
| 默认测试模型 `grok-4.5` | Grok OAuth 账号连接测试默认走可用模型 |
| `expired` → `expires_at` 规范化 | 导入 CLIProxyAPI 账号后 token 刷新逻辑可正常工作 |
| 透传 `credentials.headers` | 带上 Grok CLI 的 `User-Agent` / `x-xai-token-auth` 等头 |
| 401 / refresh 永久失败 → `error` | 吊销的 refresh token 账号不再反复被调度 |
| 403 临时踢出调度 | 权限/额度拒绝冷却 30 分钟～2 小时，不硬禁用 |

涉及文件：

```
backend/internal/handler/admin/account_handler.go
backend/internal/service/account.go
backend/internal/service/account_test_service.go
backend/internal/service/admin_account.go
backend/internal/service/grok_media.go
backend/internal/service/grok_quota_service.go
backend/internal/service/grok_token_provider.go
backend/internal/service/openai_gateway_cc_pipeline.go
backend/internal/service/openai_gateway_grok.go
backend/internal/service/openai_gateway_grok_test.go
```

---

## 快速开始（Docker，推荐）

### 1. 克隆

```bash
git clone https://github.com/dishenglee/sub2api-grok-cli.git
cd sub2api-grok-cli
```

### 2. 部署

按上游官方 Docker 文档操作（本仓完整保留 `deploy/`）：

```bash
cd deploy
cp .env.example .env
# 按需修改 .env 中的域名、管理员密码、数据库密码等
docker compose up -d --build
```

更详细说明见：

- [deploy/DOCKER.md](deploy/DOCKER.md)
- [deploy/README.md](deploy/README.md)
- 上游中文文档：[README_CN.md](README_CN.md)

### 3. 打开后台

浏览器访问你配置的域名或 `http://服务器IP:端口`，用初始化管理员账号登录。

### 4. 导入 Grok CLI 账号

支持两种来源：

1. **网页 OAuth 授权**（后台 Grok 账号 → OAuth）
2. **导入 CLIProxyAPI / Grok Shell 的 auth JSON**

CLI 账号 credentials 典型字段：

```json
{
  "type": "xai",
  "email": "you@example.com",
  "access_token": "...",
  "refresh_token": "...",
  "expired": "2026-07-13T12:00:00Z",
  "base_url": "https://cli-chat-proxy.grok.com/v1",
  "headers": {
    "User-Agent": "grok-shell/0.2.93 (linux; x86_64)",
    "x-xai-token-auth": "xai-grok-cli",
    "x-grok-client-version": "0.2.93"
  }
}
```

本 fork 会自动把 `expired` 规范成 `expires_at`，并在请求时带上 `headers`。

### 5. 调用 API

```bash
curl https://你的域名/v1/chat/completions \
  -H "Authorization: Bearer sk-你的密钥" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "grok-4.5",
    "messages": [{"role":"user","content":"你好"}]
  }'
```

---

## 本地开发构建（可选）

```bash
# 后端
cd backend
go mod download
go test ./internal/service -run Grok -count=1
go build -o sub2api ./cmd/...

# 前端
cd ../frontend
pnpm install
pnpm build
```

具体命令以仓库内 `Makefile`、`DEV_GUIDE.md` 为准。

---

## 账号状态说明（Grok）

| 上游结果 | 本 fork 行为 | 账号 status |
|---|---|---|
| 403 权限/风控 | 临时不可调度 30 分钟（订阅类 2 小时） | 仍为 `active` |
| 401 + refresh 吊销 | 标记 `error` + 长时间踢出 | `error` |
| 429 限流 | 按 rate limit 冷却 | `active`（限流中） |

---

## 和原版的关系

- **上游项目**：[Wei-Shaw/sub2api](https://github.com/Wei-Shaw/sub2api)
- **本仓库**：完整 fork + Grok CLI 兼容改动
- **许可证**：遵循上游 LICENSE
- **免责声明**：使用可能违反上游服务商条款，风险自负；仅供学习研究

上游完整功能说明、赞助商信息、详细部署步骤请继续阅读 [README_CN.md](README_CN.md)。

---

## 反馈

- 本 fork 的 Grok CLI 问题：在本仓库开 Issue  
- 上游通用功能问题：优先到 [Wei-Shaw/sub2api](https://github.com/Wei-Shaw/sub2api) 反馈  

---

## 致谢

感谢 [Wei-Shaw/sub2api](https://github.com/Wei-Shaw/sub2api) 原作者与社区。

---

> 👤 作者：涤生AGI | 🐙 github.com/dishenglee | 📡 公众号：涤生AGI

---

**LINUX DO** — a new kind of community, where tech enthusiasts gather.  
https://linux.do
