# OpenCut 后端功能与服务指南

## 1. 总体架构
OpenCut 的后端围绕 Next.js App Router 的 API Route 构建，主要负责鉴权、数据库访问、第三方服务代理与音频转写调度。核心逻辑位于 `apps/web/src/app/api/*`，同时复用 `packages/auth` 与 `packages/db` 提供的共享模块；音频转写则通过独立的 Modal Python 服务部署在 `apps/transcription` 中。【F:apps/web/src/app/api/auth/[...all]/route.ts†L1-L4】【F:packages/auth/src/server.ts†L1-L48】【F:packages/db/src/index.ts†L1-L38】【F:apps/transcription/transcription.py†L1-L143】

## 2. 身份认证与会话管理
- `packages/auth` 使用 Better Auth + Drizzle adapter 连接 Postgres，并把速率限制存入 Upstash Redis，统一导出 `auth` 实例供 Next.js API 路由复用。【F:packages/auth/src/server.ts†L1-L48】
- `/api/auth/[...all]` 直接把所有 GET/POST 请求交给 Better Auth 处理，实现登录、注册、会话续期与注销等功能，无需额外控制器代码。【F:apps/web/src/app/api/auth/[...all]/route.ts†L1-L4】
- `packages/db` 暴露懒加载的 Drizzle ORM 客户端与用户、会话、第三方账户、验证码、导出等待名单等表结构，供认证模块与其他服务共享。【F:packages/db/src/index.ts†L1-L38】【F:packages/db/src/schema.ts†L1-L70】

## 3. 数据模型与持久化
Postgres 中启用了行级安全 (RLS) 的 `users`、`sessions`、`accounts`、`verifications`、`export_waitlist` 五张表，对应账号、会话令牌、OAuth 绑定、验证码以及导出功能等待名单。所有表都在 `schema.ts` 中集中定义，可作为阅读数据库层的入口。【F:packages/db/src/schema.ts†L1-L70】

## 4. 音频上传与自动字幕流水线
1. 前端请求 `/api/get-upload-url`，后端会先做 Upstash Redis 速率限制、校验转写配置，然后用 Cloudflare R2 凭据生成 1 小时有效的预签名 URL，指导浏览器把音频文件直接上传到 R2 对象存储。【F:apps/web/src/app/api/get-upload-url/route.ts†L22-L128】
2. 上传完成后，前端调用 `/api/transcribe`，该路由同样做速率限制、参数校验，并把文件名与可选的 AES-GCM 密钥转发给 Modal 的 FastAPI 端点。【F:apps/web/src/app/api/transcribe/route.ts†L52-L188】
3. Modal 端的 `transcription.py` 在 GPU 容器中下载 R2 上的音频、按需解密、使用 Whisper `base` 模型转写，再删除对象并返回字幕段落，这一流程完全运行在 Modal 托管的 Python 环境里。【F:apps/transcription/transcription.py†L12-L137】
4. 两个路由都依赖 `isTranscriptionConfigured` 校验 Cloudflare 与 Modal 所需的环境变量，确保缺失配置时能提前返回错误，避免空跑。【F:apps/web/src/lib/transcription-utils.ts†L1-L13】【F:apps/web/src/env.ts†L7-L44】

## 5. 声音素材搜索代理
`/api/sounds/search` 暴露给前端的声音素材搜索接口：它校验查询参数、执行速率限制，然后向 Freesound Public API 发送请求，附加评分、授权、时长等过滤条件，再把结果映射为客户端需要的字段并返回。这条链路实现了对第三方声音库的服务端代理，避免在浏览器暴露密钥。【F:apps/web/src/app/api/sounds/search/route.ts†L1-L265】

## 6. 导出功能等待名单
`/api/waitlist/export` 路由负责收集用户邮箱：服务端验证邮箱格式、检查是否重复订阅，如果是新邮箱则写入 `export_waitlist` 表，否则返回 `alreadySubscribed` 标记。整个过程共用 Upstash 速率限制与 Drizzle ORM 查询/插入逻辑。【F:apps/web/src/app/api/waitlist/export/route.ts†L1-L82】【F:packages/db/src/schema.ts†L61-L70】

## 7. 运维工具与通用中间件
- `lib/rate-limit.ts` 统一封装 Upstash Redis 的滑动窗口限流器，被多个 API 路由重用，用于抵御滥用请求。【F:apps/web/src/lib/rate-limit.ts†L1-L16】
- `/api/health` 提供简单的 200 OK 健康检查，方便部署平台进行探活监控。【F:apps/web/src/app/api/health/route.ts†L1-L5】

## 8. 后端代码阅读顺序建议
1. 先阅读 `packages/db/src/schema.ts` 与 `packages/db/src/index.ts` 了解数据库连接与表结构，厘清数据模型。【F:packages/db/src/index.ts†L1-L38】【F:packages/db/src/schema.ts†L1-L70】
2. 接着查看 `packages/auth/src/server.ts` 与 `/api/auth/[...all]` 理解鉴权如何复用共享模块。【F:packages/auth/src/server.ts†L1-L48】【F:apps/web/src/app/api/auth/[...all]/route.ts†L1-L4】
3. 走查通用基础设施：`lib/rate-limit.ts`、`lib/transcription-utils.ts`、`env.ts`，掌握环境变量约束与限流组件。【F:apps/web/src/lib/rate-limit.ts†L1-L16】【F:apps/web/src/lib/transcription-utils.ts†L1-L13】【F:apps/web/src/env.ts†L7-L44】
4. 最后依次阅读业务路由：上传 (`/api/get-upload-url`)、转写 (`/api/transcribe`)、声音搜索 (`/api/sounds/search`)、等待名单 (`/api/waitlist/export`)，掌握与第三方服务的集成方式。【F:apps/web/src/app/api/get-upload-url/route.ts†L22-L128】【F:apps/web/src/app/api/transcribe/route.ts†L52-L188】【F:apps/web/src/app/api/sounds/search/route.ts†L1-L265】【F:apps/web/src/app/api/waitlist/export/route.ts†L1-L82】

