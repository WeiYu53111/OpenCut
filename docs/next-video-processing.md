# Next.js 在 OpenCut 中的视频文件处理机制

## 总览
OpenCut 的视频编辑能力完全运行在 `apps/web` 目录下的 Next.js 前端应用中。页面渲染、交互逻辑与多媒体处理都在浏览器里完成，Node.js 仅负责提供 Next.js 服务端渲染、API 代理等外围功能，因此视频编解码、抽帧、导出都依赖 Web API 与 WebAssembly 库在客户端执行。【F:apps/web/src/lib/export.ts†L150-L260】【F:apps/web/src/lib/mediabunny-utils.ts†L1-L200】

## 1. 素材导入：类型识别、元数据提取与缩略图
1. 用户在媒体面板拖拽或选择文件后，`processMediaFiles` 会对每个文件调用 `getFileType` 判断是图像、视频还是音频，并生成浏览器临时 URL。【F:apps/web/src/lib/media-processing.ts†L12-L88】【F:apps/web/src/stores/media-store.ts†L20-L74】
2. 对视频素材，前端通过 Mediabunny 的 `Input` 解析容器，提取时长、分辨率与帧率，并调用 `VideoSampleSink` 抽取关键帧绘制到 `<canvas>` 生成缩略图，无需上传到服务器。【F:apps/web/src/lib/media-processing.ts†L30-L57】【F:apps/web/src/lib/mediabunny-utils.ts†L17-L105】
3. 获取到的元数据与缩略图随后被写入 Zustand 的 `useMediaStore`，并异步保存到 IndexedDB/OPFS，以便刷新后恢复项目草稿。【F:apps/web/src/stores/media-store.ts†L88-L180】

### 学习建议
- 先阅读 `media-processing.ts` 掌握文件判别与缩略图生成，再对照 `mediabunny-utils.ts` 理解 Mediabunny 的输入、抽帧 API。
- 调试 `processMediaFiles` 的循环，观察大文件处理时的进度回调与错误提示。

## 2. 预览抽帧：Mediabunny CanvasSink + 自定义缓存
1. `videoCache` 为每个视频素材初始化 Mediabunny 的 `CanvasSink`，在前端解码并缓存帧。`getFrameAt` 根据时间戳智能复用已缓存帧或重新 seek，保持拖拽时的响应速度。【F:apps/web/src/lib/video-cache.ts†L1-L186】
2. `renderTimelineFrame` 在 Canvas 上按轨道顺序绘制时间线当前帧：针对视频元素调用 `videoCache.getFrameAt` 抽帧，针对图片使用内存缓存的 `Image` 对象，并支持背景颜色、渐变与模糊层。【F:apps/web/src/lib/timeline-renderer.ts†L1-L220】
3. 预览面板通过 `useEffect` 监听播放时间与轨道变化，优先读取帧缓存命中；若缓存未命中则使用离屏 Canvas 渲染一帧并写回缓存，同时预渲染邻近时间点以提升滑杆体验。【F:apps/web/src/components/editor/preview-panel.tsx†L468-L740】

### 学习建议
- 结合浏览器 Performance 面板观察 `videoCache` 的 seek 与 iterate 行为，理解拖拽和播放模式下的差异化处理。
- 分析 `preview-panel.tsx` 中的离屏 Canvas 与帧缓存策略，体会在 React + Canvas 结合时如何避免重复绘制。

## 3. 导出：CanvasSource + Web Audio 混音
1. 导出流程 `exportProject` 直接运行在浏览器里：创建 `<canvas>`，逐帧调用 `renderTimelineFrame` 绘制画面，并通过 Mediabunny 的 `CanvasSource` 编码为 MP4(H.264)/WebM(VP9)。【F:apps/web/src/lib/export.ts†L150-L260】
2. 音频部分使用 Web Audio API 将时间线中的音频片段解码为 `AudioBuffer`，按轨道时序合并后交给 Mediabunny 的 `AudioBufferSource`，再写入输出容器。【F:apps/web/src/lib/export.ts†L44-L148】
3. 整个导出过程由前端状态树提供轨道、媒体、项目设置，不依赖服务器转码，用户的数据始终留在本地浏览器环境中。【F:apps/web/src/lib/export.ts†L150-L260】

### 学习建议
- 跟踪导出循环中的 `renderTimelineFrame` 调用，理解时间线状态如何转化为最终帧。
- 尝试调整 `quality`、`fps` 参数，观察编码器质量映射与导出耗时的关系。

## 4. Next.js/Node.js 的职责边界
- Next.js 负责打包、路由、鉴权 API 与部署管线；视频处理核心依赖浏览器 API、Mediabunny 与 FFmpeg WebAssembly 在客户端执行。【F:apps/web/src/lib/mediabunny-utils.ts†L1-L200】
- 由于流程大量使用 `window`, `document`, Canvas 与 Web Audio，代码无法在 Node.js 环境运行，故服务器端只提供页面与接口壳层，所有媒体数据始终留在用户设备上。【F:apps/web/src/lib/export.ts†L150-L260】【F:apps/web/src/stores/media-store.ts†L20-L120】

## 推荐阅读顺序
1. `apps/web/src/stores/media-store.ts`：了解媒体文件在 Zustand 中的生命周期与本地持久化逻辑。
2. `apps/web/src/lib/media-processing.ts` 与 `apps/web/src/lib/mediabunny-utils.ts`：掌握导入阶段的元数据解析与缩略图抽取。
3. `apps/web/src/lib/video-cache.ts`、`apps/web/src/lib/timeline-renderer.ts`：理解预览帧渲染与缓存策略。
4. `apps/web/src/components/editor/preview-panel.tsx`：学习 React 组件如何调用渲染管线并协调播放。
5. `apps/web/src/lib/export.ts`：最后研究导出编码流程，完成从导入到产出的闭环。
