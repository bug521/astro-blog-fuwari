---
title: "ChatMore:多模型同时对话系统"
published: 2025-11-10
tags: ["系统设计", "Vue", "Go", "工具"]
description: "一个前后端分离的 Go + Vue 项目，可同时与多个 OpenAI 兼容模型对话并比较输出结果，支持 SSE 流式通信、会话记录持久化、JWT 登录。"
category: "项目设计"
slug: "chat-more-design-doc"
---

> 目标：一个前后端分离的 Go + Vue 项目，可同时与多个 OpenAI 兼容模型对话并比较输出结果，支持 SSE 流式通信、会话记录持久化、JWT 登录。

## 🧱 一、总体目标与功能

### 功能概述

- **多模型并行对话**：一次输入，多个兼容 OpenAI API 的模型并行响应。

- **结果对比展示**：并排显示各模型返回结果。

- **会话管理**：多轮对话记录保存与查询。

- **模型管理**：配置多个企业/提供商的模型及密钥。

- **用户系统**：用户名+密码登录，JWT 认证。

- **实时流式输出**：SSE（Server-Sent Events）通道。

- **统计与监控**：模型延迟、成功率、token 消耗汇总。

- **Docker 部署**：前后端容器化运行。

### 非目标

- 不实现 diff 结果对比。

- 不支持文件上传、向量检索、对象存储、任务队列。

- 暂不考虑多租户、多用户并发规模优化。

## 🧩 二、系统架构

```plain text
前端(Vue3 + Naive UI)  <--SSE-->  后端(Go + Gin + MySQL + sqlx)
                                    |
                                    +-- OpenAI兼容模型A
                                    +-- OpenAI兼容模型B
                                    +-- ...

```

### 架构分层

| 层级 | 职责 |
| --- | --- |
| 前端 | 用户界面、模型选择、SSE流接收、消息渲染、历史与统计展示 |
| API层 | 认证、模型管理、会话管理、SSE聚合与分发 |
| 服务层 | 并行调度、多模型调用、数据整合与入库 |
| 持久层 | MySQL（sqlx操作）、Redis（可选缓存） |
| 外部服务 | 各企业模型API（OpenAI兼容） |

## ⚙️ 三、技术栈

### 前端

- **框架**：Vue 3 + Vite + TypeScript

- **组件库**：Naive UI

- **状态管理**：Pinia

- **通信**：Axios + SSE

- **路由**：Vue Router

- **样式**：Tailwind / Naive 自带样式

- **Markdown 渲染**：markdown-it + highlight.js

### 后端

- **语言框架**：Go + Gin

- **数据库**：MySQL

- **ORM/SQL**：sqlx（不使用 GORM）

- **加密**：bcrypt（密码）、AES（API Key）

- **JWT**：RS256（Access、Refresh、Stream token）

- **配置管理**：Viper

- **日志监控**：Zap + Prometheus（可选）

- **并发控制**：errgroup + context

- **部署**：Docker Compose（mysql + api + web）

## 🗃️ 四、数据库与核心数据模型

### users

| 字段 | 类型 | 描述 |
| --- | --- | --- |
| id | bigint | 主键 |
| email | varchar | 唯一邮箱 |
| display_name | varchar | 显示名 |
| password_hash | varchar | bcrypt哈希 |
| role | enum(admin/user) | 角色 |
| failed_attempts | int | 登录失败次数 |
| locked_until | datetime | 锁定截止 |
| is_active | bool | 是否启用 |
| last_login_at | datetime | 最近登录 |
| created_at / updated_at | datetime | 时间戳 |

### refresh_tokens

| 字段 | 描述 |
| --- | --- |
| id (jti) | JWT ID |
| user_id | 用户ID |
| issued_at / expires_at | 时间 |
| revoked_at | 失效时间 |
| user_agent / ip | 设备信息 |

### models

| 字段 | 描述 |
| --- | --- |
| id | 主键 |
| name | 模型名 |
| provider_name | 提供商 |
| base_url | API地址 |
| api_key_enc | 加密密钥 |
| default_params | JSON参数(temperature/top_p/max_tokens/timeout_ms) |
| enabled | 是否启用 |
| notes | 备注 |
| created_at / updated_at | 时间戳 |

### conversations

| 字段 | 描述 |
| --- | --- |
| id | 会话ID |
| title | 标题 |
| user_id | 所属用户 |
| tags | JSON数组 |
| metadata | JSON |
| archived_at | 归档时间 |
| last_message_at | 最后消息时间 |
| created_at / updated_at | 时间戳 |

### messages

| 字段 | 描述 |
| --- | --- |
| id | 消息ID |
| conversation_id | 会话 |
| turn_id | 对话轮次 |
| role | user/system/assistant |
| model_ref | 对应模型 |
| content | JSON(OpenAI格式) |
| tokens_prompt / tokens_completion | token统计 |
| latency_ms | 延迟 |
| error | JSON |
| created_at | 时间戳 |

### turns & turn_results

- `turns`：一次提问（用户发起）

- `turn_results`：每个模型对该轮提问的独立结果（包含使用量与耗时）

## 🔐 五、用户认证系统（JWT）

### 登录机制

- 用户名+密码登录

- bcrypt 验证

- 签发：

  - **Access Token (15min)**

  - **Refresh Token (14天)** 存 HttpOnly Cookie

### 刷新机制

- `/auth/refresh` 旋转 Refresh Token

- 过期或撤销则需重新登录

### SSE 鉴权

- `POST /streams/ticket` 获取短期 **stream_token**（JWT，60秒有效）

- `GET /streams/turn/:id?st=<stream_token>` 建立 SSE

### JWT Claims

| 字段 | 含义 |
| --- | --- |
| sub | 用户ID |
| email | 用户邮箱 |
| role | 角色 |
| iat / exp | 签发与过期时间 |
| jti | 唯一标识 |

## 🔄 六、核心业务流程

### 1. 用户登录

→ `/auth/login`
→ 返回 access_token & cookie(rt)

### 2. 创建会话

→ `/api/conversations`
→ 数据库存储会话基础信息

### 3. 发送消息

→ `/api/conversations/:id/messages`
→ 后端并行调用多个模型
→ 保存 `turn` 与 `turn_results`
→ 返回 `turn_id`

### 4. 建立SSE流

→ `/streams/ticket` → 获取 stream_token
→ 前端使用 EventSource 建立连接：
`/streams/turn/:turn_id?st=token`

### 5. 接收事件流

事件类型：

- `chunk`: 增量文本

- `final`: 结束与usage

- `error`: 错误信息

- `done`: 所有模型完成

### 6. 查看历史

→ `/api/conversations/:id/messages`

### 7. 查看统计

→ `/api/stats/usage?from=...&to=...`

## 💬 七、接口概览

| 模块 | 方法 | 路径 | 说明 |
| --- | --- | --- | --- |
| Auth | POST | /auth/register | 用户注册（可配置开关） |
| Auth | POST | /auth/login | 登录 |
| Auth | POST | /auth/refresh | 刷新令牌 |
| Auth | POST | /auth/logout | 登出（当前设备） |
| Auth | GET | /auth/me | 当前用户信息 |
| Auth | POST | /auth/change-password | 修改密码 |
| Model | GET | /api/models | 模型列表 |
| Model | POST | /api/models | 新增模型 |
| Model | PATCH | /api/models/:id | 更新模型 |
| Model | POST | /api/models/:id/test | 测试连通性 |
| Conv | POST | /api/conversations | 创建会话 |
| Conv | GET | /api/conversations | 查询会话 |
| Conv | GET | /api/conversations/:id | 会话详情 |
| Msg | POST | /api/conversations/:id/messages | 发送消息并触发对话 |
| Msg | GET | /api/conversations/:id/messages | 历史消息 |
| Stream | POST | /streams/ticket | 获取SSE票据 |
| Stream | GET | /streams/turn/:id | 建立SSE流 |
| Stats | GET | /api/stats/usage | 模型统计 |

## 🔀 八、SSE 流事件格式

### 事件类型与结构

```json
// event: chunk
{ "turn_id":"t1","model_id":"m1","seq":1,"delta_text":"Hello","ts":"2025-11-10T09:00:00Z" }

// event: final
{ "turn_id":"t1","model_id":"m1","is_final":true,
  "usage":{"prompt_tokens":123,"completion_tokens":456,"total_tokens":579},
  "latency_ms":2345 }

// event: error
{ "turn_id":"t1","model_id":"m1","is_final":true,
  "error":{"code":"PROVIDER_TIMEOUT","message":"timeout"} }

// event: done
{ "turn_id":"t1","models_total":3,"models_succeeded":2,"models_failed":1 }

```

## 🧮 九、统计与监控

### 统计口径

- 按模型汇总：

  - 成功率

  - 平均延迟 / P95

  - token 使用量

### 监控

- Gin 日志 + Zap

- Prometheus metrics（请求数、错误率、延迟）

- 健康检查：`/healthz`

## 🔒 十、安全与配置

| 配置项 | 示例值 | 说明 |
| --- | --- | --- |
| JWT_ALG | RS256 | JWT算法 |
| JWT_ACCESS_TTL | 15m | Access有效期 |
| JWT_REFRESH_TTL | 336h | Refresh有效期 |
| JWT_STREAM_TTL | 60s | SSE票据有效期 |
| AUTH_ALLOW_SELF_REGISTER | false | 是否允许注册 |
| MYSQL_DSN | user:pass@tcp(mysql:3306)/chat?parseTime=true | 数据源 |
| CORS_ALLOWED_ORIGINS | https://app.localhost | 允许跨域 |
| COOKIE_SECURE | true | 仅HTTPS |
| COOKIE_SAMESITE | Lax | 防CSRF |
| BCRYPT_COST | 12 | 加密成本 |

## 🧭 十一、前端架构

```plain text
/frontend
  /src
    /views
      Login.vue
      Conversations.vue
      ConversationDetail.vue
      Models.vue
      Stats.vue
    /components
      ChatPane.vue
      MessageItem.vue
      ModelColumn.vue
      ModelSelector.vue
    /stores
      authStore.ts
      sessionStore.ts
      modelsStore.ts
      streamStore.ts
    /api
      auth.ts
      models.ts
      conversations.ts
      streams.ts
    /router
  vite.config.ts

```

### 前端交互流程

1. 登录页 → 获取 access_token + Cookie

1. 选择模型 → 创建会话

1. 输入消息 → 触发 SSE 流 → 动态渲染每个模型的回复

1. 查看历史与统计

---

## 🖥️ 十二、后端目录结构

```plain text
/backend
  /cmd/server                # 主程序入口
  /internal
    /config                  # 解析env
    /middleware             # auth, rate-limit, cors
    /handlers               # auth, models, conversations, streams, stats
    /service
      /auth                   # 登录/刷新/注销
      /conversation            # 会话逻辑
      /provider                # openai兼容调用
      /stats
    /repo                     # sqlx访问
    /security                 # jwt/bcrypt
    /sse                      # ticket管理、缓冲池
    /util                     # 错误与验证
  go.mod

```

## 🧰 十三、部署与运行

### Docker Compose 示例

```yaml
version: "3"
services:
  api:
    build: ./backend
    env_file: .env
    ports: ["8080:8080"]
    depends_on: [mysql]
  web:
    build: ./frontend
    ports: ["3000:80"]
  mysql:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: chat
    ports: ["3306:3306"]

```

## 📊 十四、错误码表

| 分类 | 代码 | 含义 |
| --- | --- | --- |
| Auth | AUTH_INVALID_CREDENTIALS | 用户名或密码错误 |
| Auth | AUTH_ACCOUNT_LOCKED | 账号暂时锁定 |
| Auth | AUTH_TOKEN_EXPIRED | 访问令牌过期 |
| Auth | AUTH_TOKEN_REVOKED | 刷新令牌失效 |
| Auth | AUTH_REGISTER_DISABLED | 注册被关闭 |
| Model | MODEL_NOT_FOUND | 模型不存在 |
| Model | PROVIDER_TIMEOUT | 调用超时 |
| Conversation | CONVERSATION_NOT_FOUND | 会话不存在 |
| Common | VALIDATION_FAILED | 参数错误 |
| Common | INTERNAL_ERROR | 服务内部错误 |

## 🧩 十五、迭代路线图

| 阶段 | 功能 |
| --- | --- |
| MVP | 登录、会话、多模型并行对话、SSE 流、持久化 |
| 第二阶段 | 模型管理界面、统计报表、限流与熔断 |
| 第三阶段 | RAG 支持、文件上传、权限与审计（未来） |

## ✅ 十六、总结

该系统通过 **Go + Vue3 + SSE** 实现高效的多模型对话聚合与展示，
具有清晰的层次结构、易于扩展的模型接入能力、可配置的认证系统，
适合作为企业内部 **模型对比与评测平台** 的基础架构。

