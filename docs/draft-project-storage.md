# 草稿项目存储格式说明

本文档汇总 OpenCut Web 端草稿项目（本地草稿）的持久化结构，并给出在浏览器外部脚本/控制台中批量生成草稿项目的步骤，方便与外部资产管线衔接。

## 总览

草稿项目完全存放在浏览器侧的 IndexedDB 与 OPFS（Origin Private File System）中：

- `video-editor-projects` 数据库的 `projects` 对象仓库存放项目元数据（序列化后的 `SerializedProject`）。【F:apps/web/src/lib/storage/storage-service.ts†L21-L103】
- 每个项目拥有独立的媒体仓库：
  - `video-editor-media-${projectId}` 数据库的 `media-metadata` 仓库存放 `MediaFileData` 元数据。【F:apps/web/src/lib/storage/storage-service.ts†L42-L189】【F:apps/web/src/lib/storage/types.ts†L12-L24】
  - `media-files-${projectId}` OPFS 目录存放真实的媒体二进制文件，键名与媒体 `id` 对应。【F:apps/web/src/lib/storage/storage-service.ts†L42-L188】【F:apps/web/src/lib/storage/opfs-adapter.ts†L1-L41】
- 时间线数据位于 `video-editor-timelines-${projectId}`（或带 `-${sceneId}` 后缀）的 `timeline` 仓库，记录 `TimelineData`（轨道列表与最后更新时间）。【F:apps/web/src/lib/storage/storage-service.ts†L55-L334】【F:apps/web/src/lib/storage/types.ts†L26-L36】

## 数据结构

### 项目 (`SerializedProject`)

`SerializedProject` 由 `TProject` 序列化而来，时间字段转化为 ISO 字符串，附带场景数组。【F:apps/web/src/lib/storage/storage-service.ts†L74-L138】【F:apps/web/src/lib/storage/types.ts†L51-L60】

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | `string` | 项目标识符。 |
| `name` | `string` | 项目名称。 |
| `thumbnail` | `string` | 缩略图 Base64 或 URL，占位可为空。 |
| `createdAt` / `updatedAt` | `string` | ISO8601 时间戳。 |
| `scenes` | `SerializedScene[]` | 场景列表，包含 `id`、`name`、`isMain`、序列化时间字段。【F:apps/web/src/lib/storage/storage-service.ts†L77-L99】【F:apps/web/src/lib/storage/types.ts†L38-L41】 |
| `currentSceneId` | `string` | 当前激活场景。 |
| `backgroundColor` / `backgroundType` / `blurIntensity` | 多个背景设置。 |
| `bookmarks` | `number[]` | 书签时间点（秒）。 |
| `fps` | `number` | 项目帧率，默认 30fps。【F:apps/web/src/stores/project-store.ts†L11-L43】 |
| `canvasSize` | `{ width: number; height: number }` | 画布尺寸。 |
| `canvasMode` | `'preset' | 'original' | 'custom'` | 画布尺寸来源。 |

生成默认草稿时，可以参考 `createDefaultProject`，其默认创建「Main Scene」、1080p 画布、黑色背景等配置。【F:apps/web/src/stores/project-store.ts†L24-L43】

### 媒体 (`MediaFileData` + OPFS 文件)

- `MediaFileData` 存储名称、类型、大小、分辨率、时长等元数据，键名为媒体 `id`。【F:apps/web/src/lib/storage/types.ts†L12-L24】
- 实际 `File` 被写入 OPFS 目录 `media-files-${projectId}`，外部脚本需通过 `navigator.storage.getDirectory()` 获取目录句柄后写入文件内容。【F:apps/web/src/lib/storage/opfs-adapter.ts†L10-L34】

### 时间线 (`TimelineData`)

`TimelineData` 的核心是 `TimelineTrack[]`，每个轨道包含元素数组。元素结构详见 `TimelineElement`：

- 媒体元素：引用媒体库 `mediaId`，包含时长、入点、出点、静音等字段。【F:apps/web/src/types/timeline.ts†L17-L43】
- 文本元素：内嵌文本样式、位置、旋转、透明度等属性。【F:apps/web/src/types/timeline.ts†L24-L40】

轨道记录 `type`（`media`/`text`/`audio`）、是否为主轨 (`isMain`) 等信息。【F:apps/web/src/types/timeline.ts†L83-L134】

## 从外部生成草稿项目

以下步骤可在浏览器控制台、Playwright、Electron 等环境执行，向现有 IndexedDB/OPFS 中注入一套完整草稿数据。

### 1. 准备项目与场景

```ts
const projectId = crypto.randomUUID();
const sceneId = crypto.randomUUID();

const now = new Date().toISOString();
const serializedProject = {
  id: projectId,
  name: "批量导入示例",
  thumbnail: "",
  createdAt: now,
  updatedAt: now,
  scenes: [
    {
      id: sceneId,
      name: "Main Scene",
      isMain: true,
      createdAt: now,
      updatedAt: now,
    },
  ],
  currentSceneId: sceneId,
  backgroundColor: "#000000",
  backgroundType: "color",
  blurIntensity: 8,
  bookmarks: [],
  fps: 30,
  canvasSize: { width: 1920, height: 1080 },
  canvasMode: "preset",
};
```

### 2. 写入项目元数据

```ts
const projectRequest = indexedDB.open("video-editor-projects", 1);
projectRequest.onupgradeneeded = () => {
  const db = projectRequest.result;
  if (!db.objectStoreNames.contains("projects")) {
    db.createObjectStore("projects", { keyPath: "id" });
  }
};
projectRequest.onsuccess = () => {
  const db = projectRequest.result;
  const tx = db.transaction("projects", "readwrite");
  tx.objectStore("projects").put({ id: projectId, ...serializedProject });
};
```

### 3. 可选：导入媒体

```ts
async function saveMediaFile({ id, name, type, blob }) {
  // 1) 写入 metadata
  const mediaDbReq = indexedDB.open(`video-editor-media-${projectId}`, 1);
  mediaDbReq.onupgradeneeded = () => {
    const db = mediaDbReq.result;
    if (!db.objectStoreNames.contains("media-metadata")) {
      db.createObjectStore("media-metadata", { keyPath: "id" });
    }
  };
  mediaDbReq.onsuccess = async () => {
    const db = mediaDbReq.result;
    const tx = db.transaction("media-metadata", "readwrite");
    tx.objectStore("media-metadata").put({
      id,
      name,
      type,
      size: blob.size,
      lastModified: Date.now(),
    });

    // 2) 写入 OPFS 文件
    const root = await navigator.storage.getDirectory();
    const dir = await root.getDirectoryHandle(`media-files-${projectId}`, {
      create: true,
    });
    const fileHandle = await dir.getFileHandle(id, { create: true });
    const writable = await fileHandle.createWritable();
    await writable.write(blob);
    await writable.close();
  };
}
```

### 4. 可选：写入时间线

```ts
const tracks = [
  {
    id: crypto.randomUUID(),
    name: "Main Track",
    type: "media",
    isMain: true,
    elements: [
      {
        id: crypto.randomUUID(),
        type: "media",
        name: "导入片段",
        mediaId: "your-media-id",
        duration: 5,
        startTime: 0,
        trimStart: 0,
        trimEnd: 5,
      },
    ],
  },
];

const timelineReq = indexedDB.open(
  `video-editor-timelines-${projectId}-${sceneId}`,
  1
);
timelineReq.onupgradeneeded = () => {
  const db = timelineReq.result;
  if (!db.objectStoreNames.contains("timeline")) {
    db.createObjectStore("timeline", { keyPath: "id" });
  }
};
timelineReq.onsuccess = () => {
  const db = timelineReq.result;
  const tx = db.transaction("timeline", "readwrite");
  tx.objectStore("timeline").put({
    id: "timeline",
    tracks,
    lastModified: new Date().toISOString(),
  });
};
```

### 5. 验证

执行完成后，刷新编辑器页面，`useProjectStore` 会从 IndexedDB 拉取项目列表、初始化场景与时间线，草稿即会出现在项目列表中。【F:apps/web/src/stores/project-store.ts†L45-L138】【F:apps/web/src/stores/timeline-store.ts†L270-L308】

> **提示**：若需批量生成，可将上述逻辑封装在自定义脚本内，循环写入多个项目与媒体。确保 `id` 不重复，并在写入结束后调用 `URL.revokeObjectURL` 等手段清理临时引用。
