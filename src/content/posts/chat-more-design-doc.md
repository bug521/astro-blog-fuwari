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

### refresh_tokens

### models

### conversations

### messages

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

## 🧩 十五、迭代路线图

## ✅ 十六、总结

该系统通过 **Go + Vue3 + SSE** 实现高效的多模型对话聚合与展示，
具有清晰的层次结构、易于扩展的模型接入能力、可配置的认证系统，
适合作为企业内部 **模型对比与评测平台** 的基础架构。

