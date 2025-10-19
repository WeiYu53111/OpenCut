# OpenCut 代码阅读指南

## 高层架构
- **单一代码库结构。** 仓库在 `apps/web` 下托管 Next.js Web 编辑器，在 `apps/transcription` 下托管基于 Modal 的转写服务，并将共享包（身份验证、数据库等）放在 `packages/` 中，供各应用复用。【F:README.md†L19-L35】【F:apps/transcription/transcription.py†L1-L143】
- **隐私优先定位。** 公共 README 强调本地编辑、无付费墙，并通过 Databuddy 处理分析，从产品目标层面为阅读代码提供背景。【F:README.md†L15-L27】

## Web 应用分层（`apps/web`）
### 应用骨架与全局提供者
- 根布局负责串联全局样式、主题、提示上下文、基于 IndexedDB/OPFS 的存储访问以及场景迁移器，同时注入分析与机器人防护脚本。【F:apps/web/src/app/layout.tsx†L1-L62】
- 全局元数据定义集中管理 OpenGraph、图标与 Manifest 配置，供各路由复用。【F:apps/web/src/app/metadata.ts†L1-L86】

### 路由概览
- 路由目录覆盖营销落地页、编辑器工作区与活动内容，例如首页体验的 `page.tsx`、项目作用域的编辑器 `editor/[project_id]/page.tsx`，以及位于 `why-not-capcut/page.tsx` 的 CapCut 对比页面。【F:apps/web/src/app/page.tsx†L1-L21】【F:apps/web/src/app/editor/[project_id]/page.tsx†L1-L200】【F:apps/web/src/app/why-not-capcut/page.tsx†L1-L160】

### 使用 Zustand 的状态管理
- `useProjectStore` 协调项目生命周期：创建默认场景、通过存储服务加载/保存、管理书签、帧率、画布尺寸，并在切换项目时同步依赖的状态分片。【F:apps/web/src/stores/project-store.ts†L1-L200】
- `useTimelineStore` 是编辑交互的核心，负责轨道排序、历史栈、波纹编辑、吸附、剪贴板操作、持久化以及与媒体/场景存储的整合。【F:apps/web/src/stores/timeline-store.ts†L1-L400】
- 其他分片管理播放计时、布局预设和场景增删改查，各自聚焦特定职责（例如基于 requestAnimationFrame 的播放循环，以及感知场景的时间线重新加载）。【F:apps/web/src/stores/playback-store.ts†L1-L174】【F:apps/web/src/stores/editor-store.ts†L1-L91】【F:apps/web/src/stores/scene-store.ts†L1-L161】

### 持久化层
- IndexedDB 适配器存储结构化元数据，OPFS 负责大型二进制数据块，存储服务协调项目、时间线、媒体与已保存音频的项目级数据库。【F:apps/web/src/lib/storage/indexeddb-adapter.ts†L1-L66】【F:apps/web/src/lib/storage/opfs-adapter.ts†L1-L58】【F:apps/web/src/lib/storage/storage-service.ts†L1-L200】

### 媒体导入与工具
- 导入编辑器的媒体文件会先通过辅助工具和 Mediabunny 封装进行类型检查、尺寸计算、缩略图生成与时长测量，再写入存储。【F:apps/web/src/lib/media-processing.ts†L1-L89】
- 时间线校验工具确保片段在轨道内不重叠，并在渲染前维持排序约束。【F:apps/web/src/lib/timeline.ts†L1-L41】

### 编辑界面
- 时间线组件通过协调 Zustand 状态、钩子与 UI 基元，将缩放、吸附、框选、上下文菜单、播放控制与拖放导入组合起来。【F:apps/web/src/components/editor/timeline/index.tsx†L1-L200】
- 预览渲染会拉取活跃轨道、媒体与项目设置，结合帧缓存，并暴露布局参考线与文本拖拽手柄，方便实时调整。【F:apps/web/src/components/editor/preview-panel.tsx†L1-L200】
- 帧缓存对活跃元素与项目设置进行哈希，以便重复渲染时复用缓存的 ImageData，即便叠加元素较多也能保持播放流畅。【F:apps/web/src/hooks/use-frame-cache.ts†L1-L198】

### 导出流程
- `exportProject` 利用 Mediabunny 的输出将时间线帧绘制到画布，混合音频缓冲，并根据所选质量与音频选项导出 MP4/WebM 文件。【F:apps/web/src/lib/export.ts†L1-L200】
- 帧渲染在复用缓存视频帧以提升性能的同时，也考虑背景虚化模式、缩放比例与媒体类型差异。【F:apps/web/src/lib/timeline-renderer.ts†L1-L120】

### 安全与集成
- 零知识加密工具生成客户端 AES-GCM 密钥，为后续需由转写工作器等服务解密的资源做准备。【F:apps/web/src/lib/zk-encryption.ts†L1-L71】
- 身份验证钩子调用共享的 Better Auth 客户端包，支持邮箱/密码与 Google 登录流程，并引导用户进入项目仪表盘。【F:apps/web/src/hooks/auth/useLogin.ts†L1-L59】【F:packages/auth/src/client.ts†L1-L8】

## 转写服务（`apps/transcription`）
- 基于 Modal 部署的 FastAPI 端点从 Cloudflare R2 下载加密音频，可选地通过 AES-GCM 解密，随后在 GPU 支持的工作器上运行 Whisper，调整时间戳并清理存储后返回转写结果。【F:apps/transcription/transcription.py†L1-L143】

## 推荐阅读顺序
1. **从产品目标入手**——阅读仓库 README，了解隐私目标、功能范围与高层结构。【F:README.md†L15-L35】
2. **理解应用骨架**——查看 `app/layout.tsx` 与 `app/metadata.ts`，了解全局提供者和元数据默认值。【F:apps/web/src/app/layout.tsx†L1-L62】【F:apps/web/src/app/metadata.ts†L1-L86】
3. **浏览关键路由**——快速查看首页 (`page.tsx`)、编辑器工作区 (`editor/[project_id]/page.tsx`) 以及营销内容 (`why-not-capcut/page.tsx`)，在深入前先建立整体印象。【F:apps/web/src/app/page.tsx†L1-L21】【F:apps/web/src/app/editor/[project_id]/page.tsx†L1-L200】【F:apps/web/src/app/why-not-capcut/page.tsx†L1-L160】
4. **审阅状态存储**——依次阅读 `project-store.ts`、`timeline-store.ts`、`playback-store.ts` 与 `scene-store.ts`，理解项目、轨道、播放与场景之间的数据流。【F:apps/web/src/stores/project-store.ts†L1-L200】【F:apps/web/src/stores/timeline-store.ts†L1-L400】【F:apps/web/src/stores/playback-store.ts†L1-L174】【F:apps/web/src/stores/scene-store.ts†L1-L161】
5. **追踪持久化适配器**——研究存储服务及其适配器，了解项目、媒体与时间线如何在本地保存。【F:apps/web/src/lib/storage/storage-service.ts†L1-L200】【F:apps/web/src/lib/storage/indexeddb-adapter.ts†L1-L66】【F:apps/web/src/lib/storage/opfs-adapter.ts†L1-L58】
6. **跟进媒体导入流程**——查看 `media-processing.ts` 与 `timeline.ts`，了解导入资源在进入时间线前如何准备与校验。【F:apps/web/src/lib/media-processing.ts†L1-L89】【F:apps/web/src/lib/timeline.ts†L1-L41】
7. **研究编辑界面**——逐步阅读时间线与预览组件及其配套钩子（如 `use-frame-cache`），理解运行时的编辑器行为。【F:apps/web/src/components/editor/timeline/index.tsx†L1-L200】【F:apps/web/src/components/editor/preview-panel.tsx†L1-L200】【F:apps/web/src/hooks/use-frame-cache.ts†L1-L198】
8. **学习导出逻辑**——阅读 `timeline-renderer.ts` 与 `export.ts`，了解播放画面如何转化为可下载视频。【F:apps/web/src/lib/timeline-renderer.ts†L1-L120】【F:apps/web/src/lib/export.ts†L1-L200】
9. **检查外部集成**——最后浏览身份验证（`useLogin`、Better Auth 客户端）、加密工具以及外部转写服务，理解应用如何与托管服务协同。【F:apps/web/src/hooks/auth/useLogin.ts†L1-L59】【F:packages/auth/src/client.ts†L1-L8】【F:apps/web/src/lib/zk-encryption.ts†L1-L71】【F:apps/transcription/transcription.py†L1-L143】
