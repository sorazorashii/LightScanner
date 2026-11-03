# LightScanner —— 类扫描全能王的 Web 文档扫描应用 · 开发文档

> 适用场景：vibe coding（自然语言驱动、AI 结对生成代码）
> 技术栈：React 18 + TypeScript + Vite + Zustand + Web Worker（Comlink）

---

## 目录

1. [设计目标与约束](#设计目标与约束)
2. [架构总览](#架构总览)
3. [组件清单与职责](#组件清单与职责)
4. [模块间通信总则（核心）](#模块间通信总则核心)
5. [通信机制一：Store —— 状态通信](#通信机制一store-状态通信)
6. [通信机制二：Command —— 动作通信](#通信机制二command-动作通信)
7. [通信机制三：EventBus —— 跨模块通知](#通信机制三eventbus-跨模块通知)
8. [通信机制四：Worker —— 重计算隔离](#通信机制四worker-重计算隔离)
9. [接口契约规范（TypeScript 先行）](#接口契约规范typescript-先行)
10. [数据流与状态迁移](#数据流与状态迁移)
11. [错误处理与错误码](#错误处理与错误码)
12. [强制规则（违规即拒合入）](#强制规则违规即拒合入)
13. [目录结构与文件命名](#目录结构与文件命名)
14. [AI 结对编码规则与提示词模板](#ai-结对编码规则与提示词模板)
15. [附录：事件 / 命令 / 错误码全集](#附录事件-命令-错误码全集)

> 所有章节标题均使用 GitHub 风格的自动生成锚点（转小写、去标点、空格转 `-`），可直接点击跳转。

---

## 1. 设计目标与约束

### 1.1 产品目标

构建一个纯前端、可离线的文档扫描 Web 应用，覆盖扫描全能王的核心链路：

**拍摄/导入 → 自动框选 → 透视矫正 → 滤镜增强 → 多页编排 → OCR → 导出 PDF**

| 目标 | 指标 |
|---|---|
| 首屏可交互 | ≤ 2s（Vite 预构建 + 代码分割） |
| 单页 8MB 图片处理 | 主线程无 jank，长任务 < 50ms |
| 离线可用 | 全链路不依赖后端，数据存 IndexedDB |
| OCR 可选 | 云端 OCR 失败时不阻断本地流程 |

### 1.2 给 AI 的约束（最重要）

> 本仓库**不是**写给人类阅读理解的代码库，而是写给 AI 反复生成的代码库。
> 因此一切设计都服务于三个字：**可校验**。

1. **契约先行**：任何新模块，先写 `types.ts` + `contract.ts`，再写实现。AI 只填实现，不设计接口。
2. **通信只走三条通道**：Store / Command / EventBus。AI 不得发明第四种通信方式（如直接 import 另一个模块的内部函数并调用其副作用）。
3. **单向数据流**：UI → Command → Service → Store → UI。禁止反向或跨层。
4. **失败可见**：所有跨模块调用必须携带 `result: Ok<T> | Err<E>`，禁止抛异常穿越模块边界。
5. **命名即文档**：事件名、命令名、store slice 名必须让 AI 仅凭名字知道方向和 payload。

### 1.3 非目标

- 不做服务端渲染、不做 SEO
- 不做多租户 / 账号体系（本地优先，云端仅为可选同步）
- 不做原生 App 壳（后续可用 Capacitor 套壳，但不影响本设计）

---

## 2. 架构总览

### 2.1 分层

```
┌──────────────────────────────────────────────┐
│  UI Layer    (React Components / Pages)      │  只负责渲染 + 派发意图
├──────────────────────────────────────────────┤
│  Facade     (useCommand / useStore hooks)    │  唯一出口，屏蔽下层复杂度
├──────────────────────────────────────────────┤
│  Command    (业务动作，有副作用、可撤销)       │  编排多个 Service，事务性
├──────────────────────────────────────────────┤
│  Service    (纯领域能力，无 UI)              │  可独立单测
├──────────────────────────────────────────────┤
│  Infra      (Store / EventBus / Worker /     │  技术细节，对上层透明
│              Persistence / Permission)       │
└──────────────────────────────────────────────┘
```

**铁律**：依赖只允许**上层依赖下层**，箭头不可逆。`Service` 永远不知道 `UI` 的存在，`UI` 永远不直接调用 `Service` 的方法。

### 2.2 为什么不用"全局 EventBus 一把梭"

| 方式 | 优点 | 致命缺点（vibe coding 下） |
|---|---|---|
| 全局事件 | 解耦 | AI 会疯狂发事件，半年后无人知道谁在监听，调试全靠全局搜索 |
| 直接调用 service | 简单 | 跨模块调用无法拦截、无法回滚、无事务 |
| **三分法（本文）** | 各司其职、可机械校验 | 需要约定判据 |

**判据（AI 每次调用前必须自问）**：

| 问题 | 归属 |
|---|---|
| 这个数据被多个组件**同时看着**、且会随时间变化？ | **Store** |
| 这是一个**动作**，有副作用、可能失败、可能要撤销？ | **Command** |
| 这件事**已经发生过了**，某个模块需要知道但不必响应？ | **EventBus** |
| 这是**重计算**，会阻塞主线程 > 16ms？ | **Worker** |

---

## 3. 组件清单与职责

### 3.1 模块地图

| 层 | 模块 | 职责 | 通信偏好 |
|---|---|---|---|
| UI | `CameraCapture` | 调用摄像头、实时预览 | 派发 Command |
| UI | `GalleryImport` | 相册/文件选取 | 派发 Command |
| UI | `FrameDetector` | 实时检测文档四边 | 订阅 Worker 结果，写 Store |
| UI | `CropEditor` | 拖拽四角、透视预览 | 双向绑定 Store |
| UI | `FilterPanel` | 滤镜选择与参数 | 派发 Command |
| UI | `PageManager` | 多页排序、增删、缩略图 | 派发 Command + 订阅 Store |
| UI | `OcrPanel` | OCR 结果展示与校对 | 订阅 Store，派发 Command |
| UI | `ExportDialog` | 导出格式与参数配置 | 派发 Command |
| Service | `ImagePipeline` | 透视矫正、裁剪、滤镜链 | 被 Command 调用，跑 Worker |
| Service | `FilterEngine` | 滤镜算法集合 | 纯函数，被 Pipeline 调 |
| Service | `OcrService` | 文字识别（本地/云端） | 被 Command 调用 |
| Service | `ExportService` | 合成 PDF / JPEG | 被 Command 调用 |
| Service | `DocumentModel` | 文档与页的领域模型 | Store 内聚 |
| Infra | `store` | 全局状态（Zustand） | 被所有层读写 |
| Infra | `commandBus` | 命令派发与中间件 | Facade 出口 |
| Infra | `eventBus` | 强类型事件广播 | 仅 Service/Infra 发射 |
| Infra | `workerPool` | Worker 生命周期与调度 | 被 Service 调 |
| Infra | `persistence` | IndexedDB 读写 | 被 Command 调 |
| Infra | `permissionGate` | 相机/存储权限 | 被 Facade 调 |

### 3.2 组件通信禁区图

```
✅ CameraCapture ──Command──▶ ImagePipeline ──写──▶ Store ──通知──▶ CropEditor
✅ ImagePipeline ──Event──▶ 任意监听者（单向、无响应）

❌ CameraCapture ──直接调用──▶ store 内部方法（绕过 Command，无审计）
❌ CropEditor ──import──▶ OcrService（跨层，UI 直连 Service）
❌ Service ──import──▶ React 组件（反向依赖）
❌ A 组件 ──Event──▶ B 组件 ──Event──▶ A 组件（事件回环）
```

---

## 4. 模块间通信总则（核心）

> **本章是全文最重要的一章。所有其他章节都是它的注脚。**
> AI 在生成任何跨模块代码前，必须完整读一遍 4.1 ~ 4.4。

### 4.1 通信三定律

**定律一：一切跨模块数据交换，必须显式声明在 `contract.ts` 中。**
禁止在任何组件内部临时拼一个对象当消息发。发什么、收什么，全在项目里搜得到。

**定律二：单向。永远单向。**
```
意图向上冒泡（UI → Facade → Command）
数据向下流动（Store → Selector → UI）
通知单向广播（Service → EventBus → 监听者，不回执）
```

**定律三：失败是一等公民。**
模块边界的每一次调用都返回 `Result<T, E>`。异常（`throw`）只允许在模块**内部**使用，跨边界必须被捕获并转化为 `Err`。

### 4.2 基础类型（全项目统一）

```ts
// src/shared/types.ts

export type Result<T, E = AppError> =
  | { ok: true; value: T }
  | { ok: false; error: E };

export const Ok = <T>(value: T): Result<T, never> => ({ ok: true, value });
export const Err = <E>(error: E): Result<never, E> => ({ ok: false, error });

export interface AppError {
  readonly code: ErrorCode;
  readonly message: string;
  readonly cause?: unknown;
  readonly retryable: boolean;
}

export type ErrorCode =
  | 'PERMISSION_DENIED'
  | 'DEVICE_UNAVAILABLE'
  | 'IMAGE_TOO_LARGE'
  | 'DETECT_FRAME_FAILED'
  | 'WORKER_TIMEOUT'
  | 'EXPORT_FAILED'
  | 'PERSISTENCE_ERROR'
  | 'OCR_UNAVAILABLE'
  | 'UNKNOWN';
```

### 4.3 通信选型决策树（AI 必读）

```
这个调用发生在两个模块之间？
 │
 ├─ 需要对方立刻返回一个值？ ──▶ 同步调用（Service 方法），返回 Result
 │
 ├─ 是一次性动作，会改状态，可能失败/撤销？ ──▶ Command
 │
 ├─ 是状态，会被多处同时读、会变化？ ──▶ Store slice
 │
 ├─ 是"事情已经发生了"的通知？ ──▶ Event（只发不答）
 │
 └─ 会占用 >16ms 计算？ ──▶ Worker RPC
```

### 4.4 禁止的通信模式

| 模式 | 为什么禁 | 替代方案 |
|---|---|---|
| 组件 `ref` 调用兄弟组件方法 | 隐式依赖，无法追踪 | 状态下沉到 Store |
| `window` 上挂全局变量传数据 | 不可类型化、不可测试 | Store / EventBus |
| 事件携带"请求"并期待回执 | 把事件变成 RPC，最难调试 | 改用 Command 或 Worker RPC |
| 在 reducer 里发事件 / 调 service | 副作用混入纯函数 | reducer 只改状态，副作用放 Command |
| 直接 `fetch` / `navigator.mediaDevices` 散落在组件 | 无法统一拦截、无法 mock | 全部封装进 Infra 层 Service |

---

## 5. 通信机制一：Store —— 状态通信

### 5.1 定位

Store **只存放**：
- 被多个组件同时消费的状态
- 需要跨会话保留的状态
- UI 的当前意图（当前选中页、当前滤镜、裁剪框坐标）

Store **不存放**：
- 可由其他状态推导的值（用 selector 计算，见 5.4）
- 一次性动作的中间结果
- 不会变化的配置

### 5.2 Slice 划分（禁止大杂烩）

每个领域一个 slice，slice 之间**不得互相 import**。

```ts
// src/store/index.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';

export const useAppStore = create<RootState>()(
  persist(
    (...a) => ({
      capture: createCaptureSlice(...a),
      document: createDocumentSlice(...a),
      editor: createEditorSlice(...a),
      export: createExportSlice(...a),
    }),
    {
      name: 'scanweb-v1',
      storage: createJSONStorage(() => idbStorage), // IndexedDB，非 localStorage
      partialize: (s) => ({ document: s.document }), // 只持久化文档数据
    }
  )
);
```

```ts
// src/store/slices/document.slice.ts
export interface DocumentSlice {
  pages: Page[];
  activePageId: string | null;

  // ✅ 只暴露 action，action 内部可走 Command 或直接改状态
  addPages: (files: File[]) => Promise<Result<void>>;
  removePage: (id: string) => void;
  setActivePage: (id: string) => void;
  updatePageImage: (id: string, blob: Blob) => void;
}

export const createDocumentSlice: StateCreator<
  RootState, [], [], DocumentSlice
> = (set, get) => ({
  pages: [],
  activePageId: null,

  addPages: async (files) => {
    // 重逻辑交给 Command，这里只做状态回填
    const result = await commandBus.execute(new AddPagesCommand(files));
    if (result.ok) set({ pages: [...get().pages, ...result.value] });
    return result;
  },
  removePage: (id) => set((s) => ({
    pages: s.pages.filter((p) => p.id !== id),
    activePageId: s.activePageId === id ? null : s.activePageId,
  })),
  setActivePage: (id) => set({ activePageId: id }),
  updatePageImage: (id, blob) => set((s) => ({
    pages: s.pages.map((p) => p.id === id ? { ...p, image: blob } : p),
  })),
});
```

### 5.3 同步 vs 异步规则

| 操作 | 放哪 |
|---|---|
| 纯状态变更（切换选中页、改滤镜参数） | slice 内同步 action |
| 涉及 IO / Worker / 可能失败 | slice action 内调用 Command，拿 Result 回填 |
| 跨 slice 协作 | 在 Command 里通过 `get()` 读状态、统一 `set` 多 slice |

### 5.4 Selector 必须派生，禁止冗余状态

```ts
// ✅ 正确：派生
export const selectActivePage = (s: RootState) =>
  s.document.pages.find((p) => p.id === s.document.activePageId) ?? null;

export const selectPageCount = (s: RootState) => s.document.pages.length;

// ❌ 错误：冗余字段 activePage 对象，与 pages 不同步必然出 bug
```

### 5.5 组件只通过 hook 订阅，禁止直接 `useAppStore()`

```ts
// src/store/hooks.ts
import { useShallow } from 'zustand/react/shallow';

export const useActivePage = () =>
  useAppStore(useShallow(selectActivePage));

export const usePageList = () =>
  useAppStore(useShallow((s) => s.document.pages));

// ❌ 禁止：useAppStore((s) => s.document) 会把整个 slice 当依赖，任意字段变化都重渲染
```

---

## 6. 通信机制二：Command —— 动作通信

### 6.1 为什么要有 Command

Store action 适合"改状态"，但**拍摄一张照片**这个动作需要：
1. 请求相机权限（可能失败）
2. 捕获帧（可能失败）
3. 派 Worker 做帧检测（可能超时）
4. 把结果写入 Store
5. 若第 3 步失败，需要回滚第 2 步的临时状态

这是**事务性动作**，Reducer 模式表达不了。Command 模式正好。

### 6.2 接口契约

```ts
// src/shared/command.ts
export interface Command<TResult = void> {
  readonly name: string;        // 全局唯一，用于日志与中间件
  readonly reversible: boolean; // 是否支持 undo
  execute(): Promise<Result<TResult>>;
  undo?(): Promise<Result<void>>;
}

export interface CommandBus {
  execute<T>(cmd: Command<T>): Promise<Result<T>>;
  registerMiddleware(mw: CommandMiddleware): void;
}

export interface CommandMiddleware {
  handle<T>(cmd: Command<T>, next: () => Promise<Result<T>>): Promise<Result<T>>;
}
```

### 6.3 一个完整 Command 示例

```ts
// src/commands/AddPagesCommand.ts
export class AddPagesCommand implements Command<Page[]> {
  readonly name = 'document/add-pages';
  readonly reversible = true;

  constructor(private readonly files: File[]) {}

  async execute(): Promise<Result<Page[]>> {
    // 1. 校验
    const valid = this.files.filter((f) => f.type.startsWith('image/'));
    if (valid.length === 0) return Err(makeErr('IMAGE_INVALID', '无有效图片'));

    // 2. 调 Service（走 Worker）
    const detectResult = await imagePipeline.detectAndWarp(valid);
    if (!detectResult.ok) return detectResult; // 错误沿 Result 传播，不 throw

    // 3. 构造领域对象
    const pages: Page[] = detectResult.value.map((img) => ({
      id: crypto.randomUUID(),
      image: img.blob,
      quad: img.quad,       // 检测到的四角
      filter: 'original',
      createdAt: Date.now(),
    }));

    // 4. 持久化（失败可重试）
    const saveResult = await persistence.savePages(pages);
    if (!saveResult.ok && saveResult.error.retryable) {
      return Err(saveResult.error); // 调用方（slice）可选择重试
    }

    return Ok(pages); // slice 收到后回填 store
  }

  async undo(): Promise<Result<void>> {
    // 撤销 = 删除刚才创建的页
    return persistence.removePagesByIds(/* ... */);
  }
}
```

### 6.4 中间件（可观测性全靠它）

```ts
// src/infra/commandBus.ts
const bus: CommandBus = {
  execute: async (cmd) => {
    const start = performance.now();
    logger.debug(`[cmd:start] ${cmd.name}`);
    const result = await runWithMiddlewares(cmd);
    logger.debug(`[cmd:end] ${cmd.name} ${result.ok ? 'ok' : 'fail'} ${(performance.now()-start).toFixed(1)}ms`);
    // 所有命令的结果都发一条事件，供 telemetry 与调试面板订阅
    eventBus.emit({ type: 'command.executed', payload: { name: cmd.name, ok: result.ok } });
    return result;
  },
};
```

### 6.5 给 AI 的铁律

- **每个 Command 必须有唯一 `name`，格式 `{领域}/{动作}`**，如 `export/build-pdf`、`ocr/recognize`
- Command 内部可以调多个 Service，但**禁止**直接 `store.setState`（必须通过返回值让 slice 回填，保证事务原子性）
- Command **禁止**发射"请求类"事件，它本身就是请求

---

## 7. 通信机制三：EventBus —— 跨模块通知

### 7.1 定位

事件 = **"事情已经发生"**。发射者不关心谁在听，也不等回执。
用途：生命周期通知、跨模块解耦的副作用、遥测、调试。

### 7.2 强类型契约（核心防错手段）

**禁止 string 事件名 + any payload。** 全项目事件在 `event-map.ts` 一张表定义，TypeScript 会替你校验每个 `emit` 和每个 `on` 的 payload 类型。

```ts
// src/shared/event-map.ts
export interface EventMap {
  // 领域事件
  'document.page-added':      { pageIds: string[] };
  'document.page-removed':    { pageIds: string[] };
  'document.all-cleared':     {};

  // 处理进度（瞬态，丢失不影响正确性）
  'pipeline.progress':        { pageId: string; stage: Stage; ratio: number };
  'pipeline.done':            { pageId: string; elapsedMs: number };

  // 系统事件
  'system.permission-changed': { kind: 'camera' | 'storage'; granted: boolean };
  'system.online':            {};
  'system.offline':           {};
  'system.error':             { code: ErrorCode; message: string };

  // 遥测（由 commandBus 中间件统一发射）
  'command.executed':         { name: string; ok: boolean };
}

// 由此派生出类型安全的 bus
export type EventType = keyof EventMap;
export type PayloadOf<T extends EventType> = EventMap[T];
```

```ts
// src/infra/eventBus.ts
export interface TypedEventBus {
  emit<T extends EventType>(event: { type: T; payload: PayloadOf<T> }): void;
  on<T extends EventType>(type: T, handler: (p: PayloadOf<T>) => void): Unsubscribe;
  once<T extends EventType>(type: T, handler: (p: PayloadOf<T>) => void): Unsubscribe;
}

// 实现要点：
// - emit 是同步、无回执、无异常外泄（handler 内部错误被 catch 并 logger.error）
// - handler 抛出不影响其他 handler
// - 组件卸载必须 unsubscribe，用 useEffect 的 cleanup 保证
```

### 7.3 事件的三个纪律

**纪律一：事件是事实，不是请求。**
```
✅ 'document.page-added'     —— 页已加好了
❌ 'document.request-add'    —— 别用事件去"请求"做事
```

**纪律二：payload 是不可变快照，不是引用。**
事件一旦发出，payload 不得再变。发射前先结构化克隆：
```ts
emit({ type: 'document.page-added', payload: { pageIds: [...ids] } });
```

**纪律三：禁止事件回环。**
A 监听 `page-added` 后不得再触发会导致 `page-added` 的逻辑。需要联动时改用 Command 编排。

### 7.4 UI 订阅事件的标准写法

```ts
useEffect(() => {
  const off = eventBus.on('pipeline.progress', ({ pageId, ratio }) => {
    if (pageId === activePageId) setRatio(ratio);
  });
  return off; // 必写，否则每次重渲染都叠一层监听
}, [activePageId]);
```

---

## 8. 通信机制四：Worker —— 重计算隔离

### 8.1 为什么必须隔离

图像处理（透视变换、滤镜卷积、OCR 前处理）单次可达数百毫秒。跑在主线程会让相机预览卡顿、手势掉帧，在 vibe coding 产出的"不讲究性能"的代码里尤其致命。

### 8.2 架构：主线程只发 RPC，不碰像素

```
┌────────────┐   Comlink    ┌─────────────────────────┐
│ Main       │ ───────────▶ │ ImageWorker              │
│ Command    │  async rpc   │  - detectEdges()         │
│            │ ◀─────────── │  - warpPerspective()     │
│            │  Blob /      │  - applyFilter()          │
│            │  ImageBitmap │  - runOcr()               │
└────────────┘              └─────────────────────────┘
                              使用 OffscreenCanvas + 纯 JS 算法
```

### 8.3 通信契约

```ts
// src/workers/image.worker.ts (被 Worker 加载的入口)
import * as Comlink from 'comlink';

const api = {
  async detectEdges(bitmap: ImageBitmap): Promise<Quad | null> { /* ... */ },
  async warpPerspective(bitmap: ImageBitmap, quad: Quad): Promise<Blob> { /* ... */ },
  async applyFilter(blob: Blob, filter: FilterName): Promise<Blob> { /* ... */ },
};
Comlink.expose(api);
export type ImageWorkerApi = typeof api;
```

```ts
// src/infra/workerPool.ts —— 主线程侧
export class WorkerPool {
  private workers: Comlink.Remote<ImageWorkerApi>[] = [];

  async call<T>(method: keyof ImageWorkerApi, args: unknown[], signal?: AbortSignal): Promise<Result<T>> {
    const w = this.pick();
    try {
      const res = await Promise.race([
        (w[method] as any)(...args),
        new Promise((_, rej) => signal?.addEventListener('abort', () => rej(new Error('abort')))),
      ]);
      return Ok(res);
    } catch (e) {
      return Err(makeErr('WORKER_TIMEOUT', String(e), true));
    }
  }
}
```

### 8.4 四条硬规则

1. **主线程永远不 `import` 图像处理算法**。`FilterEngine` 的实现文件必须标记 `/** @bundle-target worker-only */`，构建时校验。
2. **传输用 `ImageBitmap` / `Blob` / `Transferable`**，禁止传 `HTMLCanvasElement`（不可转移）。
3. **每个 RPC 必须可取消**：Command 层传 `AbortSignal`，用户切页时立刻取消上一个页的滤镜计算。
4. **Worker 抛错必须包成 `Result`**，主线程不得 `try/catch` 裸异常。

### 8.5 超时与降级

```ts
const res = await workerPool.call<Blob>('applyFilter', [bitmap, filter], signal);
if (!res.ok && res.error.code === 'WORKER_TIMEOUT') {
  eventBus.emit({ type: 'system.error', payload: { ...res.error, message: '滤镜处理超时，已跳过' } });
  return fallbackOriginal(bitmap); // 降级而非阻断
}
```

---

## 9. 接口契约规范（TypeScript 先行）

### 9.1 每个模块的 `contract.ts` 是真相源

```
src/
  features/
    capture/
      contract.ts      ← 对外暴露什么（types + interface）
      index.ts         ← 仅 re-export contract 中的公开 API
      Capture.tsx      ← 实现
      hooks.ts         ← 实现
```

**规则**：其他模块**只能** `import` 自 `contract.ts` 与 `index.ts`。直接 import 实现文件视为依赖泄漏。

### 9.2 契约文件模板

```ts
// src/features/capture/contract.ts
import type { Result } from '@/shared/types';
import type { AppError } from '@/shared/types';

// 1. 对外类型（export）
export interface CapturedFrame {
  readonly blob: Blob;
  readonly quad: Quad | null;   // null = 自动检测失败，走手动框选
  readonly width: number;
  readonly height: number;
}

export interface CaptureService {
  open(): Promise<Result<void>>;
  capture(): Promise<Result<CapturedFrame>>;
  close(): Promise<void>;
}

// 2. 注入令牌（依赖倒置，便于测试与 mock）
export const CaptureServiceToken = Symbol('CaptureService');

// 3. 不 export 实现类、不 export 内部状态类型
```

### 9.3 命名与格式约定

| 项 | 约定 | 示例 |
|---|---|---|
| 类型/接口 | `PascalCase` | `CapturedFrame` |
| 属性/变量 | `camelCase` | `activePageId` |
| 常量 | `UPPER_SNAKE` | `MAX_IMAGE_BYTES` |
| 事件类型 | `kebab-case` 带领域前缀 | `pipeline.progress` |
| Command 名 | `{领域}/{动词}` | `document/add-pages` |
| slice 文件名 | `*.slice.ts` | `document.slice.ts` |
| 契约文件名 | `contract.ts` | 固定 |
| 布尔属性 | 肯定式，`is/has/should` 开头 | `isProcessing` ❌ `processing`（易误用） |

### 9.4 向后兼容契约

契约一旦被其他模块 import，**不得**：
- 删除字段
- 收窄字段类型（如 `string` → `'a'|'b'`）
- 改事件 payload 结构

需要变更时：**新增字段 + 旧字段标记 `@deprecated` 一个版本后再删**。

---

## 10. 数据流与状态迁移

### 10.1 主流程：从拍摄到导出

```
用户点「拍摄」
   │
   ▼
CameraCapture ──▶ useCommand('capture.frame')         [Facade hook]
   │
   ▼
CaptureCommand.execute()
   ├─▶ permissionGate.ensure('camera')  ──Result──▶ 失败 → Err → UI 提示
   ├─▶ CaptureService.capture()         ──Result──▶ 拿到 CapturedFrame
   ├─▶ Worker.detectEdges(frame)        ──Result──▶ 失败 → 回退 quad=null（手动框选）
   └─▶ return Ok(frame)
   │
   ▼
slice.addPages(result.value) ──setState──▶ Store.pages 更新
   │
   ▼
Store ──selector──▶ PageManager / CropEditor 重渲染
   │
   ▼
eventBus.emit('document.page-added') ──▶ 缩略图缓存、遥测监听器
```

### 10.2 状态机：编辑态页面

每个 Page 在处理期间有明确的生命周期，禁止用散落布尔值表达：

```ts
export type PageStatus =
  | 'idle'              // 已导入，未处理
  | 'detecting'         // Worker 正在检测边角
  | 'needs-review'      // 自动检测失败，等用户手选四角
  | 'processing'        // 正在矫正/滤镜
  | 'ready'             // 可导出
  | 'error';            // 处理失败，可重试

// ✅ 派生：是否可导出
export const isExportable = (s: PageStatus) => s === 'ready' || s === 'idle';
```

### 10.3 乐观更新与回滚

```
UI 立即显示"处理中"  ← 乐观
  │
  ├─ Command 成功 → 用真实结果替换乐观态
  │
  └─ Command 失败 → 回滚到上一态 + 发射 system.error 事件
```

Command 的 `reversible` 字段就是为回滚准备的，**所有写 Store 的 Command 都应实现 undo**。

---

## 11. 错误处理与错误码

### 11.1 分层错误处理策略

| 层 | 职责 |
|---|---|
| Infra（Worker / Permission / Fetch） | 把原生异常（DOMException、TypeError）**翻译**成 `AppError` |
| Service | 聚合、补充上下文，透传 `Result` |
| Command | 决定重试 / 降级 / 中止，返回最终 `Result` |
| UI | 读 `Result`，映射到文案与交互（重试按钮、引导去设置） |

### 11.2 错误码语义表

| 码 | 触发场景 | retryable | UI 建议 |
|---|---|---|---|
| `PERMISSION_DENIED` | 相机/存储权限被拒 | 是（引导设置） | 弹窗引导去系统设置 |
| `DEVICE_UNAVAILABLE` | 无摄像头 / 设备占用 | 是 | 提示切换设备或重试 |
| `IMAGE_TOO_LARGE` | 单图超 32MB | 否 | 提示压缩后重试 |
| `DETECT_FRAME_FAILED` | 边角检测失败 | 是 | 自动降级手动框选 |
| `WORKER_TIMEOUT` | 计算超时 | 是 | 提示重试，建议降分辨率 |
| `EXPORT_FAILED` | PDF 合成失败 | 是 | 重试，建议减少页数 |
| `PERSISTENCE_ERROR` | IndexedDB 写入失败 | 是（含 quota 提示） | 提示清理空间 |
| `OCR_UNAVAILABLE` | 云端 OCR 不可达 | 是 | 提示稍后重试，不影响本地流程 |
| `UNKNOWN` | 兜底 | 否 | 上报 + 通用提示 |

### 11.3 错误上报

```ts
// 所有未被吞掉的错误，统一走这里
eventBus.emit({ type: 'system.error', payload: { code, message, cause } });
// 监听器负责：console + 远端上报（抽样）+ 触发 UI 提示
```

**铁律**：禁止 `catch(e){}` 空捕获。至少 `logger.warn` 或发射事件。

---

## 12. 强制规则（违规即拒合入）

> 本章不是建议，是 CI 门禁。`scripts/check-contracts.ts` 会机械化校验每一条。

### 12.1 依赖方向

- [ ] 无 UI 文件 import 自 `src/features/**/contract.ts` 之外的实现文件
- [ ] 无 Service / Infra 文件 import 自 `src/features/**/*.tsx` 或任何 React 相关
- [ ] 无 slice 文件 import 自 Command（反向）；Command 可 import slice 类型，但不得直接 setState
- [ ] 无跨 slice 直接引用（`document.slice` 不得 import `export.slice` 的内部）

### 12.2 通信

- [ ] 无组件使用 `ref` 调用兄弟方法传递业务数据
- [ ] 无 `window.__` 全局变量传数据
- [ ] 事件名全部出自 `EventMap`，无字符串字面量事件
- [ ] 事件 payload 全部为 `PayloadOf<T>` 类型
- [ ] 无事件处理器抛出未捕获异常（emit 内部兜底）
- [ ] 无 reducer / setter 内部调用 service 或 emit 事件

### 12.3 类型与健壮性

- [ ] 无 `any`（`eslint@typescript-eslint/no-explicit-any` 报错即失败）
- [ ] 无模块边界的 `throw`（全部转化为 `Result`）
- [ ] 所有 `fetch` / `mediaDevices` / `IndexedDB` 调用集中在 `src/infra/`
- [ ] 所有图像处理算法入口仅在 Worker 侧可解析（构建时断言）

### 12.4 可观测

- [ ] 每个 Command 都有唯一 `name`
- [ ] 所有 Command 执行都发出 `command.executed` 事件
- [ ] 所有用户可触发的失败路径都发出 `system.error` 事件

---

## 13. 目录结构与文件命名

```
scanweb/
├── index.html
├── vite.config.ts
├── tsconfig.json
├── scripts/
│   ├── check-contracts.ts      # 12 章规则的机械化校验
│   └── gen-event-map.ts        # 由 EventMap 生成 docs 附录片段
├── src/
│   ├── main.tsx
│   ├── App.tsx
│   ├── shared/
│   │   ├── types.ts            # Result / AppError / ErrorCode
│   │   ├── command.ts          # Command / CommandBus 接口
│   │   ├── event-map.ts        # EventMap（唯一事件注册表）
│   │   └── index.ts
│   ├── infra/
│   │   ├── store/
│   │   │   ├── index.ts
│   │   │   ├── hooks.ts
│   │   │   └── slices/
│   │   │       ├── document.slice.ts
│   │   │       ├── capture.slice.ts
│   │   │       ├── editor.slice.ts
│   │   │       └── export.slice.ts
│   │   ├── commandBus.ts
│   │   ├── eventBus.ts
│   │   ├── workerPool.ts
│   │   ├── persistence.ts      # IndexedDB
│   │   ├── permissionGate.ts
│   │   └── logger.ts
│   ├── workers/
│   │   ├── image.worker.ts     # Comlink 入口
│   │   └── algorithms/         # 纯算法，禁止 import UI
│   ├── commands/
│   │   ├── AddPagesCommand.ts
│   │   ├── CaptureFrameCommand.ts
│   │   ├── ApplyFilterCommand.ts
│   │   ├── BuildPdfCommand.ts
│   │   └── index.ts
│   ├── services/               # 无 UI 的领域能力
│   │   ├── CaptureService.ts
│   │   ├── ImagePipeline.ts
│   │   ├── FilterEngine.ts
│   │   ├── OcrService.ts
│   │   └── ExportService.ts
│   └── features/
│       ├── capture/
│       │   ├── contract.ts
│       │   ├── index.ts
│       │   ├── CameraCapture.tsx
│       │   └── FrameDetector.tsx
│       ├── editor/
│       │   ├── contract.ts
│       │   ├── CropEditor.tsx
│       │   └── FilterPanel.tsx
│       ├── document/
│       │   ├── contract.ts
│       │   ├── PageManager.tsx
│       │   └── Thumbnail.tsx
│       ├── ocr/
│       │   ├── contract.ts
│       │   └── OcrPanel.tsx
│       └── export/
│           ├── contract.ts
│           └── ExportDialog.tsx
```

**命名死规则**：
- `*.slice.ts` 只能是状态
- `*.service.ts` 只能是能力、无 UI
- `*.command.ts` 只能是动作编排
- `contract.ts` 只能包含类型、接口、Symbol 令牌

---

## 14. AI 结对编码规则与提示词模板

> 这一章是给"和你一起写代码的 AI"读的。把它的内容粘进你的 `AGENTS.md` 或 `.cursorrules`。

### 14.1 系统级规则（AI 必须遵守）

```
1. 动手写任何跨模块代码前，先定位或创建对应的 contract.ts，确认接口签名。
2. 新增一个跨模块动作 → 新建 Command，不要往 slice 里塞副作用。
3. 新增一个被多处消费的状态 → 放进对应 slice，并写一个 selector，不要加冗余字段。
4. 新增一个"已发生"的通知 → 先在 event-map.ts 加一行，再 emit / on。
5. 任何超过 16ms 的计算 → 放 src/workers/algorithms/，主线程只做 RPC 调用。
6. 模块边界的所有调用返回 Result<T, AppError>，禁止 throw 跨边界。
7. 禁止 any。拿不准类型时，用 unknown 再收窄。
8. 每个新文件都要能回答：它属于哪一层？谁可以 import 它？谁不可以？
```

### 14.2 用户需求 → 落地路径（决策模板）

AI 接到需求后，按此顺序填空：

```
需求：用户可以在滤镜面板上实时预览 5 种滤镜效果

Step 1 定位层：
  - 新状态？ → editor.slice: activeFilter, filterPreviewBlob
  - 新动作？ → ApplyFilterCommand（调 Worker）
  - 新事件？ → pipeline.progress / pipeline.done（已有，无需新增）
  - 新契约？ → FilterEngine 的 contract 补 getPresets(): FilterPreset[]

Step 2 接口先行：先改 contract.ts / event-map.ts（如有）

Step 3 实现顺序：Service → Command → slice → hook → UI 组件

Step 4 验证：该改动是否违反第 12 章任一条？运行 scripts/check-contracts.ts
```

### 14.3 可直接复用的提示词模板

**模板 A：新增一个功能模块**

```
在 src/features/<name>/ 下新增 <功能名>。

要求：
1. 先写 contract.ts：导出对外类型、Service 接口、Symbol 令牌。
2. 实现类放在同目录 <Name>.tsx（UI）或 src/services/<name>.ts（能力）。
3. 若涉及状态，在 src/infra/store/slices/ 加 <name>.slice.ts，并通过 hooks.ts 暴露 selector。
4. 若涉及副作用或 IO，在 src/commands/ 写 <Name>Command.ts，走 commandBus。
5. 若需通知其他模块，先在 src/shared/event-map.ts 声明事件，再 emit。
6. 所有跨模块返回值用 Result<T, AppError>，不用 throw。
7. 最后确认：没有 any、没有跨层 import、没有在 reducer 里调 service。
```

**模板 B：让 AI 审查一次改动是否合规**

```
审查以下 diff 是否符合 ScanWeb 通信规范（开发文档第 4、12 章）。

重点检查：
- [ ] 是否引入了新的通信方式（未走 Store/Command/EventBus/Worker 之一）
- [ ] 是否有跨层调用（UI 直连 Service、Service 引用 UI）
- [ ] 是否有 any / 未处理的 throw
- [ ] 事件名是否都在 EventMap 中
- [ ] Command 是否直接 setState
- [ ] Worker 是否通过 Transferable 传数据

列出问题，并给出符合规范的改写版本。不要只描述，给出代码。
```

**模板 C：定位 bug**

```
这个 bug 的表现是：<现象>。

请按以下顺序排查，不要猜：
1. 用 eventBus 日志确认事件是否发出（搜索 'command.executed' 与 'system.error'）
2. 用 selector 打印当前 Store 快照，确认状态是否符合预期
3. 检查是否有组件订阅事件后忘了 unsubscribe（useEffect cleanup）
4. 检查 Worker RPC 是否超时或被 abort
5. 若涉及异步，确认 slice 是否用 Result 回填而非乐观覆盖

输出根因和最小修复 diff。
```

---

## 15. 附录：事件 / 命令 / 错误码全集

> 本附录是**速查表**，也是 `event-map.ts` / `commands/index.ts` 的镜像。
> 用 `scripts/gen-event-map.ts` 可从源码自动生成本章，保持单一真相源。

### 15.1 事件全集（EventMap）

| 类型 | Payload | 发射方 | 典型监听方 | 语义 |
|---|---|---|---|---|
| `document.page-added` | `{pageIds:string[]}` | Document slice | Thumbnail、telemetry | 页已入库 |
| `document.page-removed` | `{pageIds:string[]}` | Document slice | Thumbnail 缓存清理 | 页已删除 |
| `document.all-cleared` | `{}` | Document slice | 全部面板重置 | 文档清空 |
| `pipeline.progress` | `{pageId,stage,ratio}` | ImagePipeline | 进度条 UI | 瞬态进度 |
| `pipeline.done` | `{pageId,elapsedMs}` | ImagePipeline | 性能埋点 | 处理完成 |
| `system.permission-changed` | `{kind,granted}` | permissionGate | Capture UI 状态 | 权限变化 |
| `system.online` / `system.offline` | `{}` | infra 监听 | 同步队列、提示条 | 网络变化 |
| `system.error` | `{code,message,cause?}` | 任意层 | 全局提示、上报 | 错误兜底 |
| `command.executed` | `{name,ok}` | commandBus 中间件 | 遥测、调试面板 | 命令审计 |

### 15.2 命令全集

| name | 输入 | 返回 | reversible | 说明 |
|---|---|---|---|---|
| `capture.frame` | `{}` | `Result<CapturedFrame>` | 否 | 打开相机并抓帧 |
| `document/add-pages` | `File[]` | `Result<Page[]>` | 是 | 导入+检测+持久化 |
| `document/remove-pages` | `string[]` | `Result<void>` | 是 | 删除页 |
| `editor/set-quad` | `{pageId,quad}` | `Result<void>` | 是 | 手动/自动设四角 |
| `editor/apply-filter` | `{pageId,filter}` | `Result<Blob>` | 是 | 滤镜（走 Worker） |
| `ocr/recognize` | `{pageId}` | `Result<OcrResult>` | 否 | OCR，可降级本地 |
| `export/build-pdf` | `ExportOptions` | `Result<Blob>` | 否 | 合成 PDF |
| `export/build-image` | `{pageId,format}` | `Result<Blob>` | 否 | 单页导出 |

### 15.3 错误码全集

| code | retryable | 触发方 | 用户文案建议 |
|---|---|---|---|
| `PERMISSION_DENIED` | 是 | permissionGate | 请在设置中开启权限 |
| `DEVICE_UNAVAILABLE` | 是 | CaptureService | 未检测到摄像头 |
| `IMAGE_TOO_LARGE` | 否 | AddPagesCommand | 图片超过 32MB |
| `DETECT_FRAME_FAILED` | 是 | ImagePipeline | 自动识别失败，请手动框选 |
| `WORKER_TIMEOUT` | 是 | workerPool | 处理超时，可降低分辨率重试 |
| `EXPORT_FAILED` | 是 | ExportService | 导出失败，请减少页数后重试 |
| `PERSISTENCE_ERROR` | 是 | persistence | 存储空间不足 |
| `OCR_UNAVAILABLE` | 是 | OcrService | 文字识别暂不可用，本地处理不受影响 |
| `UNKNOWN` | 否 | 兜底 | 未知错误，已上报 |

### 15.4 Store Slice 与关键 Selector

| slice | 关键状态 | 关键 selector |
|---|---|---|
| `capture` | `isOpen, stream, error` | `selectIsCapturing` |
| `document` | `pages, activePageId` | `selectActivePage`, `selectPageCount` |
| `editor` | `activeFilter, quadDraft` | `selectEditorState` |
| `export` | `format, quality, progress` | `selectExportOptions` |

### 15.5 分层依赖矩阵

| 谁调谁 | UI | Facade | Command | Service | Infra |
|---|---|---|---|---|---|
| **UI** | — | ✅ | ❌ | ❌ | ⚠️ 仅 selector/hook |
| **Facade** | — | — | ✅ | ❌ | ✅ |
| **Command** | ❌ | ❌ | — | ✅ | ✅ |
| **Service** | ❌ | ❌ | ❌ | — | ✅ |
| **Infra** | ❌ | ❌ | ❌ | ❌ | — |

> ⚠️ UI 只能读 Store（selector），不得直接写；写一律走 Facade/Command。

---

*文档结束。变更本文件时，同步更新第 15 章速查表与对应源码契约文件。*
