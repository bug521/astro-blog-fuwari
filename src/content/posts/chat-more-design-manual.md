---
title: "ChatMore：新版设计"
published: 2025-11-17
tags: ["系统设计", "Vue", "Go"]
description: "一个多会话聚合工具"
category: "项目设计"
slug: "chat-more-design-manual"
---

promor

编写一个同时与多个大模型进行分窗口对话的应用，前端分多个多个窗口，使用openai格式的接口进行对话，多个窗口第一次使用相同的输入，后续可以分开输入，默认继续使用同一个输入。后端主要进行相关对话记录，或者转发openai格式接口，便于进行记录。前端使用vue,naive-ui,后端使用go,gin,mysql,sqlx，其他的自行按照优秀实践进行补充。为我进行项目设计，不需要写代码。

先从整体设计说起，我分成这几块来讲：**整体架构 → 核心业务概念 → 前端设计（Vue + Naive UI）→ 后端设计（Go + Gin + MySQL + sqlx）→ 数据库结构 → 关键交互流程 → 后续可扩展点**。全程不写代码，只做架构 & 方案设计。

---

## 一、整体目标 & 架构

**目标：**

- 一个 Web 应用，**同时与多个大模型分窗口对话**；

- 第一次输入：**同一条用户输入广播到多个窗口 / 多个模型**；

- 后续输入：

  - 默认仍然“用同一条输入发给所有窗口”，

  - 但每个窗口可以单独编辑、单独发送（分叉对话）；

- 后端用统一的 **OpenAI Chat Completions 格式** 转发至不同模型（或者不同服务地址），并负责：

  - 对话记录存储；

  - 调用日志 & 统计；

  - 模型配置管理。

**技术栈：**

- 前端：Vue3 + Vite + TypeScript + Naive UI + Pinia（或其他状态管理）

- 后端：Go + Gin + sqlx + MySQL

- 鉴权：简单可以用 JWT + Cookie / LocalStorage

- 其他建议：

  - 日志：zap / logrus

  - 配置：viper

  - 依赖注入：wire（可选，按你习惯）

  - 部署：Docker + docker-compose（可选）

**基本架构：**

浏览器（Vue） ⇄ Gin REST API
Gin 服务 → Service 业务层 → Repository（sqlx） → MySQL
Gin 服务 → LLM Client（OpenAI 格式）→ 各个模型服务（可配置 base_url + key）

---

## 二、核心业务概念建模

你这个系统建议抽象出以下几个核心概念：

1. **User（用户）**

  - 登录用户，用于划分会话、统计配额等（即使初期只有你自己，也建议预留）。

1. **ChatSession（会话）**

  - 类似一个“项目/房间”，里面有多个窗口，每个窗口对应一个模型或模型配置。

  - 比如「代码评审比较」会话，包含 GPT-4、DeepSeek、Claude 三个窗口。

1. **ChatWindow（窗口）**

  - 一个会话下的一个“面板”，绑定一个具体的大模型配置（或某种变体）。

  - 每个窗口有自己的消息链（上下文）。

1. **ModelConfig（模型配置）**

  - 抽象“一个可以通过 OpenAI 接口访问的模型”：

  - 前端用 /models 接口拉取列表，在创建会话时勾选要用哪些模型。

1. **Message（消息）**

  - 属于某个会话 + 某个窗口；

  - role: system / user / assistant；

  - content: 文本（可以后续支持多模态）；

  - 额外字段：tokens、latency_ms、error 等便于统计。

1. **BroadcastInput（广播输入控制）**

  - 概念上是一种“输入模式”：

  - 支持标记：该窗口本次是否使用“全局输入”，还是“自定义输入”。

---

## 三、前端设计（Vue + Naive UI）

### 1. 页面结构

整体布局可以是：

- 顶部：应用标题、用户信息、全局设置（主题、语言等）

- 左侧 Sidebar：

  - 会话列表（ChatSession List）

  - “新建会话”按钮（可打开模态框选择模型）

- 右侧主区域：

  - 上方：当前会话标题、模型列表、全局控制（输入模式开关：广播 / 单窗口）

  - 中间：**多窗口栅格布局**（2~4 列，根据窗口数和屏幕宽度自适应）

  - 下方（可选）：全局输入框区域（用于一次输入广播到多个窗口）

### 2. 主要组件划分

1. `AppLayout`

  - 顶部导航 + 左侧会话列表 + 主内容区

1. `SessionList`

  - 展示会话列表，可搜索、删除、重命名

  - 点击切换当前会话

1. `ChatBoard`

  - 某个会话内部的多窗口区域

  - 接收 `sessionId` 作为 prop

1. `ChatWindow`

  - 单个模型窗口组件：

1. `GlobalInputBar`

  - 会话级别的输入框

  - 包含：

1. `SessionConfigModal`

  - 新建 / 编辑会话时配置：

### 3. 状态管理（Pinia）

可以设计一个 `useChatStore`，包含：

- 状态：

  - 当前用户信息

  - `sessions`: { id, title, windows: [...] } 简略信息

  - `currentSessionId`

  - `windowsBySession`: 基于 sessionId 的窗口结构

  - `messagesByWindow`: windowId → Message[]（按时间排序）

  - `globalInputText`

  - 每个 window 的 `windowInputText` + `useGlobalInput` 标志

  - 请求状态：每个 window 是否 loading、错误信息

- 行为：

  - `loadSessions()` / `createSession()` / `deleteSession()`

  - `loadSessionDetail(sessionId)`

  - `sendMessageToWindows({ sessionId, windowIds, content })`

  - `appendMessages(windowId, messages)`

  - `toggleUseGlobalInput(windowId, bool)`

> 注意： 第一次对话广播逻辑可以在前端做，也可以完全交由后端的 Session 状态字段控制，前端只负责简单地“发送一次给所有窗口”。

### 4. 请求 & 流式输出

- **请求形式**：

  - 建议前端只调用你自己的 REST API（如 `/api/chat/send`），不直接调用 OpenAI；

  - 是否流式（SSE/流式 chunk）：

---

## 四、后端设计（Go + Gin + MySQL + sqlx）

### 1. 项目结构（建议）

示例（不写具体代码，只讲目录职责）：

- `cmd/server/main.go`

  - 入口：初始化配置、DB、路由、启动服务

- `internal/config`

  - 从环境变量 / 配置文件读取 DB、日志、模型 API 等配置

- `internal/router`

  - 定义 Gin 路由、绑定 Handler

- `internal/handler`（或 `api`）

  - HTTP Handler 层：参数校验、调用 service、封装响应

- `internal/service`

  - 业务逻辑层：

- `internal/repository`

  - 封装对 MySQL 的操作（使用 sqlx）：

- `internal/model`

  - 数据库实体 & 领域模型结构体（与 JSON API DTO 分开更清晰）

- `internal/llm`

  - LLM 客户端抽象：

- `internal/middleware`

  - 鉴权（JWT）、请求日志、请求 ID、CORS、恢复 panic 等

### 2. 核心后端流程

### a) 创建会话

- 请求：`POST /api/sessions`

  - Body:

- 逻辑：

  1. 校验模型 ID 合法；

  1. 创建 ChatSession 记录；

  1. 为每个模型创建一个 ChatWindow；

  1. 返回 sessionId + 每个 window 的 id、模型信息。

### b) 获取会话详情

- `GET /api/sessions/:id`

  - 返回：

### c) 发送消息（广播 & 单窗口）

- `POST /api/sessions/:id/messages`

  - Body:

- 逻辑：

  1. 校验 session 属于当前用户；

  1. 确定实际要发送的 window 列表：

  1. 对于每个 window：

  1. 返回结构：

> 可以做成“并发调用多个模型”，再聚合结果返回。注意控制超时和失败重试。

### d) 获取窗口消息历史

- `GET /api/windows/:id/messages`

  - 支持分页：

  - 用于前端滚动加载。

### e) 模型配置管理

- `GET /api/models`：列出可用模型

- `POST /api/models` / `PUT /api/models/:id`（可选，仅管理员）

---

## 五、数据库结构设计（MySQL + sqlx）

下面用文字描述表结构（不写 SQL）：

1. **users**

  - id (PK)

  - email

  - password_hash（可选）

  - nickname

  - created_at

  - updated_at

1. **model_configs**

  - id (PK)

  - name（展示名，如 “GPT-4.1”）

  - provider（"openai" / "local" / "azure" …）

  - api_base（如 https://api.openai.com/v1）

  - model（openai 接口的 model 名）

  - api_key_enc（加密后的 key 或引用）

  - extra_config（JSON，可记录温度、top_p、max_tokens 默认值等）

  - enabled（是否启用）

  - created_at / updated_at

1. **chat_sessions**

  - id (PK)

  - user_id (FK → users)

  - title

  - description（可选）

  - initial_broadcast_done（bool，记录“是否已完成首次广播”，根据你的逻辑需要）

  - created_at / updated_at

1. **chat_windows**

  - id (PK)

  - session_id (FK → chat_sessions)

  - model_config_id (FK → model_configs)

  - name（窗口自定义名称，如 "GPT4 精确"）

  - sort_order（排序）

  - created_at / updated_at

1. **messages**

  - id (PK)

  - session_id (FK → chat_sessions)

  - window_id (FK → chat_windows)

  - role（"user" / "assistant" / "system"）

  - content（TEXT / LONGTEXT）

  - tokens_used（int，可选）

  - latency_ms（int，可选）

  - error_msg（若调用失败）

  - created_at

  - 索引建议：

> 如需“同一条广播输入对应多个窗口”的逻辑追踪，可以增加：

  - `broadcast_group_id` 字段，用于把一次广播中的多条 user 消息关联起来，但非必须。

---

## 六、关键交互&业务细节

### 1. “第一次相同输入 + 后续默认同输入”逻辑

可以这样设计：

- **后端维度：**

  - `chat_sessions.initial_broadcast_done` 初始 false；

  - 当收到会话第一条用户消息（该会话还没有任何 user 消息时）：

- **后续输入：**

  - 前端 UI 默认使用“全局输入模式”：

  - 若用户在某个 ChatWindow 里编辑自己的输入并点击“仅发送此窗口”：

- **这样实现：**

  - 无论前端是否严格控制，后端也保证“第一次一定是多窗口同输入”。

### 2. 上下文拼接策略

- 每个窗口维护独立上下文，即使第一条相同，后续单独分叉不会互相污染；

- 调用模型时：

  - 从 messages 表里读取该窗口最近 N 条对话（按时间排序）；

  - 若你想支持 system prompt，可以在会话或窗口级别记录 system_message，并在调用时插到最前。

### 3. 错误与超时处理

- 每个窗口调用模型时单独控制超时（例如 20~30 秒）；

- 若某个模型失败：

  - 前端该窗口显示错误消息（比如一条 role=assistant + “[调用失败：xxx]”）；

  - 其他窗口正常显示；

- 后端记录 error_msg 和调用时间，方便你后面做统计和调优。

---

## 七、后续可扩展点

1. **多租户 / Workspace 概念**

  - users 加上 organization_id

  - chat_sessions 归属到组织，便于后续给团队使用

1. **配额 & 计费**

  - 单独建 usage 表，记录每次调用 tokens 与费用；

  - 对接 Stripe / 内部结算。

1. **预设 Prompt 模板**

  - 在会话创建的时候，可选择模板（例如：代码评审、英文润色等）；

  - 将模板以 system message 的形式注入每个窗口。

1. **导出 & 分享**

  - 导出某个会话为 Markdown / JSON；

  - 生成只读分享链接（需做权限控制）。

1. **审计 & 检索**

  - 对 messages 做全文索引（ElasticSearch / OpenSearch / MySQL FULLTEXT）；

  - 支持“在所有对话中搜索关键字”。

