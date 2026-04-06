# CFChat

CFChat 是一个基于 Cloudflare 平台构建的轻量级实时聊天系统，包含登录鉴权、公开群组、私有群组、私信、文件上传，以及后台管理能力。项目采用前后端分离结构：前端使用 Vue 3 + Vite，服务端运行在 Cloudflare Workers，实时消息分发依赖 Durable Objects，数据存储使用 D1 / KV / R2。

这个项目适合用来实现团队内部沟通、轻量社区聊天，或者作为 Cloudflare 原生实时应用的参考实现。

## 核心能力

- 用户登录、会话校验、退出登录、修改密码
- 用户资料维护，支持头像上传
- 会话列表聚合展示：公开群组、私有群组、私信
- 公开群组加入、私有群组邀请、群成员管理
- 基于 WebSocket 的实时消息收发
- 支持文本消息和附件消息
- 后台管理能力：用户管理、群组管理、消息检索、会话巡检
- 消息保留期清理，支持定时软删除过期消息

## 技术栈

### 前端

- Vue 3
- Vue Router
- Vite

### 服务端

- Cloudflare Workers
- Hono
- Durable Objects

### 云资源

- D1：业务数据存储
- KV：登录会话存储
- R2：附件文件存储

## 项目架构

项目由两个主要部分组成：

- `frontend/` 负责单页应用界面、路由、状态初始化、接口请求和 WebSocket 连接
- `worker/` 负责 API、鉴权、数据库访问、实时消息房间、附件上传和定时任务

整体请求流转大致如下：

1. 前端通过 `/api/auth/*` 完成登录与会话恢复。
2. 聊天页通过 `/api/bootstrap` 拉取用户、群组、私信概览。
3. 进入某个会话后，前端通过 `/api/messages` 获取历史消息。
4. 前端连接 `/api/ws/:kind/:id`，由 `ChannelRoom` Durable Object 接管实时通信。
5. 附件上传通过 `/api/upload` 写入 R2，消息元数据写入 D1。
6. 管理端通过 `/api/admin/*` 进行用户、群组、消息巡检与维护。

## 目录结构

```text
cfchat/
├─ frontend/                  # Vue 前端应用
│  ├─ index.html              # Vite 入口 HTML
│  ├─ vite.config.js          # 前端构建配置
│  ├─ dist/                   # 构建产物目录
│  └─ src/
│     ├─ main.js              # 前端启动入口
│     ├─ App.vue              # 根组件
│     ├─ router.js            # 路由与权限跳转
│     ├─ store.js             # 会话初始化与登录态管理
│     ├─ api.js               # REST API 封装
│     ├─ ws.js                # WebSocket 连接封装
│     ├─ styles.css           # 全局样式
│     ├─ components/ui/       # 基础 UI 组件
│     └─ pages/               # 页面级组件
│        ├─ LoginPage.vue
│        ├─ ChatPage.vue
│        ├─ SettingsPage.vue
│        ├─ AdminPage.vue
│        ├─ AdminUsersPage.vue
│        ├─ AdminMessagesPage.vue
│        ├─ AdminSitePage.vue
│        └─ AdminRoomPage.vue
├─ worker/                    # Cloudflare Worker 服务端
│  ├─ schema.sql              # D1 初始化表结构
│  ├─ migrations/             # 数据库迁移脚本
│  └─ src/
│     ├─ index.js             # Worker 入口，注册 API 与定时任务
│     ├─ auth.js              # 密码哈希、会话创建、管理员判断
│     ├─ middleware.js        # 鉴权与管理员权限中间件
│     ├─ db.js                # 数据访问与会话权限校验
│     ├─ utils.js             # 通用工具函数
│     ├─ api/                 # 业务 API 路由
│     │  ├─ admin.js
│     │  ├─ channels.js
│     │  ├─ dm.js
│     │  ├─ messages.js
│     │  └─ upload.js
│     └─ do/                  # Durable Objects
│        ├─ ChannelRoom.js    # 单个会话的实时连接与广播
│        └─ Scheduler.js      # 定时清理任务
├─ package.json               # 项目脚本与依赖
└─ wrangler.toml              # Cloudflare Worker / D1 / KV / R2 配置
```

## 关键模块说明

### `frontend/src/pages/ChatPage.vue`

聊天主界面。负责：

- 会话列表展示
- 进入群组或私信
- 加载历史消息
- 发送文本和附件
- 群成员管理与邀请
- 维持实时连接状态

### `worker/src/index.js`

服务端总入口。负责：

- 挂载登录、会话、用户、群组、私信、上传、后台 API
- 注册全局鉴权中间件与管理员中间件
- 转发 WebSocket 连接到 Durable Object
- 处理定时清理任务

### `worker/src/do/ChannelRoom.js`

实时通信核心。每个会话由一个 Durable Object 实例承载，负责：

- 校验当前用户是否有权进入会话
- 接收前端 WebSocket 消息
- 写入 D1 消息记录
- 向当前房间内所有连接广播新消息

### `worker/src/api/admin.js`

后台管理接口集合，提供：

- 用户增删改与重置密码
- 群组与私信统计总览
- 按条件检索消息
- 按房间查看聊天记录

## 运行与部署

### 安装依赖

```bash
npm install
```

### 本地构建前端

```bash
npm run build
```

### 本地开发前端

```bash
npm run dev:frontend
```

### 初始化数据库

```bash
npm run d1:apply
```

### 部署到 Cloudflare

```bash
npm run deploy
```

## 环境与配置说明

项目主要配置位于 `wrangler.toml`：

- `ADMIN_USERNAMES`：哪些用户名拥有后台权限
- `MESSAGE_RETENTION_DAYS`：消息保留天数
- `ALLOWED_FILE_TYPES`：允许上传的文件类型前缀
- `MAX_FILE_SIZE`：允许上传的最大文件大小

同时需要在 Cloudflare 上正确绑定以下资源：

- D1 数据库：`DB`
- KV 命名空间：`SESSIONS`
- R2 存储桶：`FILES`
- Durable Objects：`CHANNEL_ROOM`、`SCHEDULER`

## 数据模型概览

`worker/schema.sql` 当前包含以下核心表：

- `users`：用户账号、密码哈希、显示名称、头像、禁用状态
- `channels`：公开群组、私有群组、私信会话
- `channel_members`：会话成员关系及角色
- `messages`：消息内容、附件信息、发送时间、软删除标记

## 适合继续扩展的方向

- 消息已读状态
- 撤回与编辑消息
- 多媒体消息预览
- 更细粒度的后台审计能力
- 更完整的多环境部署配置
