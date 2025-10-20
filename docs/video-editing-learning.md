# OpenCut 视频编辑实现与学习路线指南

## 分析过程
- 先阅读仓库根目录的 README，确认项目目标（隐私优先、无水印、时间线编辑）与整体目录布局，便于框定需要深入的模块。
- 根据《OpenCut 代码阅读指南》中推荐的顺序，重点抽取与视频编辑相关的 Zustand 状态存储、媒体处理与导出逻辑，并做逐文件记录。
- 结合时间线渲染、导出实现，梳理从素材导入、轨道排布、播放预览到最终产出的完整链路，明确关键调用关系。
- 最后整理学习路线与配套知识点，指出理解编辑器所需的前置概念与建议练习方向。

## 视频编辑实现概览
### 常见疑问：是否依赖其他项目？
- 视频编辑、合成与导出均在 `apps/web` 的 Next.js 前端内完成，核心逻辑直接使用浏览器 `window`、`document`、Canvas 与 Web Audio API，因此无法在 Node.js 或独立后端项目中运行。【F:apps/web/src/stores/playback-store.ts†L1-L160】【F:apps/web/src/lib/timeline-renderer.ts†L1-L160】【F:apps/web/src/lib/export.ts†L1-L160】
- 仓库中的 Node/Bun 代码仅用于启动 Next.js 应用、提供鉴权与转写等外围 API；时间线编辑与渲染不会调用这些后端模块。【F:packages/auth/src/server.ts†L1-L160】【F:apps/web/src/app/api/transcribe/route.ts†L1-L160】

### 项目结构与核心职责
- `apps/web` 是前端编辑器主体，Next.js 负责页面骨架与路由，Zustand 存储跨组件状态。
- `packages/` 存放身份认证等跨应用共享逻辑，支撑项目管理、登录等外围能力。

### 状态管理如何驱动编辑体验
- `useProjectStore` 维护当前项目的帧率、画布尺寸、场景以及与本地存储的同步；当切换项目时会级联刷新媒体与时间线数据，确保上下文一致。【F:apps/web/src/stores/project-store.ts†L1-L200】
- `useTimelineStore` 是编辑器的核心：管理轨道数组、历史记录、吸附设置、波纹编辑、元素增删改、拖拽分割、音视频分离与粘贴板操作；持久化时会调用存储服务同步 IndexedDB/OPFS。【F:apps/web/src/stores/timeline-store.ts†L1-L200】【F:apps/web/src/stores/timeline-store.ts†L200-L400】
- `usePlaybackStore` 使用 `requestAnimationFrame` 驱动播放时间更新，根据项目帧率对播放终点做帧级补偿，并通过自定义事件协调预览视频元素同步。【F:apps/web/src/stores/playback-store.ts†L1-L160】

### 媒体导入与存储
- 媒体导入流程利用工具函数检测文件类型、计算尺寸与缩略图，并记录在媒体存储中；时间线校验工具在添加元素时检查轨道重叠与排序，避免时间轴冲突。【F:apps/web/src/lib/media-processing.ts†L1-L89】【F:apps/web/src/lib/timeline.ts†L1-L41】
- 持久化层由 IndexedDB、OPFS 适配器与 `storageService` 组成，负责在本地保存项目元数据、时间线结构与媒体二进制数据，保证刷新后仍可恢复草稿。【F:apps/web/src/lib/storage/storage-service.ts†L1-L200】

### 预览与导出链路
- 预览画面由 `renderTimelineFrame` 遍历轨道，按时间过滤可见元素并自上而下绘制；对视频元素会利用 `videoCache` 抽帧，对背景支持颜色、渐变与模糊覆盖。【F:apps/web/src/lib/timeline-renderer.ts†L1-L160】
- 导出流程 `exportProject` 读取时间线状态，使用 Mediabunny 将帧绘制到 Canvas，再通过 Web Audio API 混合音频缓冲，最终编码为 MP4/WebM；导出选项可配置质量、帧率与是否包含音频。【F:apps/web/src/lib/export.ts†L1-L160】

## 学习路线
1. **理解产品目标与目录布局**：阅读 `README.md` 与《代码阅读指南》建立全局认知。【F:README.md†L15-L63】【F:docs/code-reading-guide.md†L1-L120】
2. **掌握全局状态协作**：按顺序阅读 `project-store.ts`、`timeline-store.ts`、`playback-store.ts`，画出状态依赖图帮助理解场景切换与播放更新。【F:apps/web/src/stores/project-store.ts†L1-L200】【F:apps/web/src/stores/timeline-store.ts†L1-L200】【F:apps/web/src/stores/playback-store.ts†L1-L160】
3. **追踪媒体与时间线数据流**：结合 `media-processing.ts`、`timeline.ts` 与存储服务，动手模拟一个素材导入到轨道的全过程。【F:apps/web/src/lib/media-processing.ts†L1-L89】【F:apps/web/src/lib/timeline.ts†L1-L41】【F:apps/web/src/lib/storage/storage-service.ts†L1-L200】
4. **调试预览渲染**：在浏览器调试 `renderTimelineFrame` 与相关组件，观察帧缓存、吸附、波纹编辑对渲染结果的影响。【F:apps/web/src/lib/timeline-renderer.ts†L1-L160】
5. **实验导出流程**：阅读 `export.ts`，尝试修改导出质量或音频混合策略，通过对比输出验证理解。【F:apps/web/src/lib/export.ts†L1-L160】
6. **扩展外围功能**：进一步探索零知识加密、Better Auth 集成与转写服务，理解项目如何扩展到协作与 AI 能力。【F:apps/web/src/lib/zk-encryption.ts†L1-L71】【F:packages/auth/src/client.ts†L1-L80】【F:apps/transcription/transcription.py†L1-L143】

## 学习建议与关键知识
- **前端状态管理**：熟悉 Zustand 的 `create` API、选择器与中间件，掌握如何保持状态不可变与实现撤销/重做历史栈。【F:apps/web/src/stores/timeline-store.ts†L1-L200】
- **时间线算法**：理解元素重叠检测、波纹编辑、吸附等时间线操作背后的数据结构，建议用伪数据编写单元测试验证逻辑。【F:apps/web/src/lib/timeline.ts†L1-L41】
- **浏览器多媒体 API**：深入 Canvas、Web Audio、MediaSource 等 API，理解抽帧缓存、音频混音与导出编码的性能考量。【F:apps/web/src/lib/timeline-renderer.ts†L1-L160】【F:apps/web/src/lib/export.ts†L1-L160】
- **本地持久化**：掌握 IndexedDB 与 OPFS 的读写模式，理解如何在前端实现大型媒体的分层存储与事务管理。【F:apps/web/src/lib/storage/storage-service.ts†L1-L200】
- **安全与服务集成**：了解客户端加密流程与身份认证、转写服务如何交互，为未来扩展云端渲染或协作功能打基础。【F:apps/web/src/lib/zk-encryption.ts†L1-L71】【F:packages/auth/src/client.ts†L1-L80】【F:apps/transcription/transcription.py†L1-L143】

