# 详情页多 Tab 卡死/转圈/无响应 —— 根因分析与修复方案

> 状态：**已实施（P0–P4 前端 + P2 后端 A+C）** — 代码仓提交 `4e7bedda`（前端）、`ebdd54e3`（后端，Gitee origin）
> 影响页面：基座模型详情、数据集详情（及同类“Tab 状态写入 URL query”的详情页）
> 关联：`docs/plan/ui/list-ui-ux-conventions.md`

## 一、现象
详情页（基座模型 / 数据集）含多个 Tab，**有时点击 Tab 像被锚定、切换不了，页面转圈、失去响应**；快速连点更严重。

## 二、根因（主因：整页重挂载）

`maas-web/src/layouts/MainLayout.vue:34` 以 **query 参与组件 key**：

```vue
<component :is="Component" :key="appStore.locale + '|' + $route.fullPath" />
```

而详情页切 Tab 会**写 URL query**：

| 页面 | 位置 | 写入 |
|---|---|---|
| 基座模型详情 | `components/basemodel/detail/useBaseModelDetail.ts:209` | `router.replace({ query: {...query, tab} })` |
| 基座模型详情（扫描深链） | 同上 `:565` | `router.replace({ query: {...query, variant} })` |
| 数据集详情 | `views/datasets/DatasetDetailView.vue:147` | `router.replace({ query })` |

`$route.fullPath` 含 query → key 变化 → **详情组件被销毁重建**：

1. 新实例 `onMounted` 重新 `load()`：
   - 基座模型：`loading=true` → `SkeletonBlock`（转圈）；
   - 数据集：串行 `await load() → loadVersions() → loadImports()`（`DatasetDetailView.vue:413-419`），最坏 3×超时。
2. 滚动位置被重置（`router.replace` + `scrollTo` + `DetailTabs.scrollIntoView`），表现为“被锚定”；
3. SSE 重连、轮询定时器重建；
4. **旧实例已发出的请求不会被取消**，与新请求叠加。

佐证：`composables/useQueryState.ts:17` 注释明确写道——“与 `MainLayout` 的 `:key="$route.fullPath"` 配合：query 变化即重挂载”。该设计**面向列表页**（筛选/分页写入 query，靠重挂载从 query 恢复），被详情页误用。

> 副作用：基座模型详情 `watch(() => route.query.tab)`（`useBaseModelDetail.ts:211`）在重挂载下几乎不会生效（实例已被销毁），实际切 Tab 靠“重挂载 + `readTab()`”，进一步印证设计错配。

## 三、次要放大因素：轮询

| 位置 | 间隔 | 机制 | in-flight 守卫 |
|---|---|---|---|
| 列表 `useAutoRefresh` | 10s（默认）/ 2s（有 running 且 SSE 未连） | setTimeout 链 | ✅ 完成后再排 |
| 详情 `usePolling` | 3s | setTimeout 链 | ✅ |
| **基座模型详情 `pullTimer`** | 3s | **setInterval** | ❌ 无守卫（SSE 连通时降到 1/5=15s） |
| `useResourceDetail`(autoMs) | 传入 | setInterval | ❌ |
| `FileBrowser` 自动刷新 | `refreshMs` | setInterval | ❌（仅判 hidden/preview） |
| `usePullerMetrics` | 5s | setInterval | ❌ |
| `useGpuWake` | 5s | sleep 循环 | ✅ 串行 |
| SSE 心跳（后端） | 15s | `EventStreamController.java:170` | —（`SseEmitter(0L)` 不超时） |

全局 HTTP 客户端：`api/client.ts:37` `timeout: 15000`（所有请求默认 15s）。

### 组合效应
切 Tab → 整页重挂载 → 串行 `load()`；后端慢（接近 15s）时骨架转圈；同时旧请求未取消 + 3s 无守卫轮询 + SSE 断线重连 → **浏览器每域 ~6 个连接被占满**（开发态经 Vite `localhost:5173` 单域代理，SSE 常驻占用其一）→ 新 Tab 内容请求排队 → “点不动/无响应”。快速连点成倍放大。

## 四、问题回答（对照需求）
- **有无自动刷新**：有，多处（见上表）。
- **有无同步调用**：无同步 XHR；但存在 `onMounted` **串行 await** 链与阻塞式 await，慢请求占住并发槽。
- **有无长超时（15s）**：全局 `client.ts:37` = 15s；SSE 无超时。

## 五、修复方案（分级，按优先级）

### P0 根因：query 变化不再重挂载详情页（推荐先做）
- 路由增加 `meta.queryKey`；key 改为：
  `appStore.locale + '|' + (route.meta.queryKey ? $route.fullPath : $route.path)`
- 仅对**依赖“query 重挂载”恢复状态的列表页**置 `meta.queryKey = true`：
  `ComponentsView`、`NodesView`、`EvalView`、`ImagesView`（使用 `useQueryState`），并核查 `TrainingView`（手工读写 query）。
- 详情页（基座模型/数据集等）默认 `path` key：切 Tab / `?variant=` 变纯本地状态，复用既有 `watch(route.query.*)`；不再整页重建、不再重新 `load()`。
- 兼容性：`useQueryState` 的列表页行为**完全不变**（仍 fullPath）；详情页移除“写 query 触发重挂载”的隐式依赖。
- 备选方案 A（改动更大）：全局 key 去掉 query，`useQueryState` 内部改为 `watch(route.query)` 主动同步，列表页不再依赖重挂载。回归面更大，不首选。

### P1 轮询治理
- 基座模型详情 `pullTimer`：`setInterval` → `usePolling`（setTimeout 链），或加 in-flight 守卫；
- **SSE `connected` 时完全停**兜底轮询（不再 1/5 降频）；
- `FileBrowser` / `useResourceDetail` / `usePullerMetrics`：加守卫或改 `usePolling`；
- 统一“不可见即暂停”（多数已具备）。

### P2 超时分级
- `client.ts` 保留 15s 默认；对已知慢接口（图片/数据集 estimate、dataset validate、asset scan summary、list variants）按请求放大到 30–60s，或后端改异步（202 + 任务态 + 轮询/SSE）。

### P3 详情加载
- `onMounted` 串行 `await` 改并行 `Promise.all`（数据集详情 `load/loadVersions/loadImports`）；
- 静默刷新不得置骨架（已有 `silent`，继续遵循）。

### P4 兜底
- `DetailTabs.scrollIntoView` 限制在 Tab 轨道内，避免整页跳动。

## 六、验收标准
- 切换详情页 Tab：**无整页骨架**、无滚动重置、无新增 `load()` 请求（Network 面板校验）。
- 存在进行中任务时，SSE 连通则**无兜底轮询**；断开才降级轮询且不叠加（dev 下观察在途请求数 ≤ 常量）。
- 慢接口场景（人为延迟 >10s）下连点 Tab 仍可即时切换，不出现“卡死/转圈”。
- 列表页筛选/分页/返回前进行为不回退。

## 七、影响面与回归清单
- 路由：`router/index.ts`（新增 meta）；`MainLayout.vue`（key 计算）。
- 详情页：`BaseModelDetailView`/`DatasetDetailView`（依赖 query 的内部 watch 需可用）。
- 列表页：`Components/Nodes/Eval/Images/Training`（确认仍按 query 恢复）。
- 轮询：`useBaseModelDetail.ts`、`FileBrowser.vue`、`useResourceDetail.ts`、`usePullerMetrics`。
- 首页/仪表盘、pull-tasks 等其它含 SSE 的页面（无回归）。

## 八、实施记录（已完成，提交 `4e7bedda`）
- **P0**：`router/index.ts` 新增 `meta.queryKey`（列表页 `training/eval/nodes/components/images` 置 `true`）；`MainLayout.vue` view key 改为
  `locale + '|' + (meta.queryKey ? route.fullPath : route.path)`；`useQueryState.ts` 注释同步。
  验证：切详情 Tab 不再重拉详情、无整页骨架、URL `?tab=` 正常；列表页深链（`/images?tab=builds`、`/training?tab=failed`）仍激活。
- **P1**：`useBaseModelDetail.ts` 拉取兜底轮询 `setInterval(3s, 无守卫)` → `usePolling`（setTimeout 链）且 `shouldPoll = 有进行中拉取 && !SSE.connected`，`watch(stream.connected)` 触发 `sync()`；`FileBrowser.vue`/`useResourceDetail.ts`/`usePullerInfo.ts`(usePullerMetrics) 增加在途守卫。
- **P2（仅前端）**：`api/datasets.ts`（validate 60s、upload 120s、estimate 30s）、`api/baseModels.ts`（variants/scan-summary/trigger 30s、replay 60s）、`api/filePreview.ts`（content/archive/parquet 60s、url 30s）按请求宽限超时；默认仍 15s。后端异步化（202+任务态）暂缓。
- **P3**：`DatasetDetailView.vue` 首屏 `loadVersions` 与 `loadImports` 改 `Promise.all` 并行。
- **P4**：`DetailTabs.vue` 激活项滚动由 `scrollIntoView` 改为仅滚动 `.detail-tabs__track`。
- **P2 后续修复（提交 `84bf7824`）**：异步校验无 SSE 事件，此前 `syncing` 且 SSE 连通时轮询/快刷被禁用，页面会停留「正在导入…」不刷新；
  改为「`syncing` 且无进行中导入（即校验中）时始终轮询/快速刷新；导入仍以 SSE 为主、仅断线兜底」。
- **P2 后续修复（提交 `75d6eef3`）**：数据集列表行内校验后手动 `fetchList` 不重排自动刷新定时器，快刷可能被延迟到默认 10s；
  `onValidate` 后调用 `auto.reschedule()`，验证列表 3s 内快刷（实测 gap≈3022ms）。
- **验证**：`npm run typecheck`、`i18n:audit`（18 语种）、容器 `npm run build` 均通过；Playwright 断言全绿（详见提交信息）。
- **已完成（后续）**：`TrainingView` 手工 query 逻辑迁移到 `useQueryState`（提交 `316cb4d1`）。

## 九、P2 后端异步化实施记录（已完成 A+C，提交 `ebdd54e3`）
- **A 校验异步**：`dataset-service` 新增 `AsyncConfig`（`@EnableAsync` + 有界 `ThreadPoolTaskExecutor` + 租户 `TaskDecorator`）与 `DatasetValidationRunner`；
  `DatasetUploadService` 的 `upload/validate` 改为「落库后提交、事务提交后异步执行校验」，请求立即返回 `validating`，前端轮询至 `available/invalid`。
  开关 `maas.dataset.validate.async`（默认 `true`）；集成测试置 `false` 走原同步路径（因测试在未提交事务内，异步线程看不到未提交数据）。
- **C 估算治理**：`DatasetEstimateClient` / `HfEstimateClient` / `MsEstimateClient` 增加 TTL 缓存（成功 10min、失败 30s），降低慢外部调用对 HTTP 线程的占用。
- **验证**：`dataset-service` 31 用例、`base-model-service` 32 用例全绿；运行时 E2E：上传/重校验即时返回 `validating`→轮询至 `available`（行数=3）；
  估算二连击缓存命中（`call2 ≤ 0.01s`）。已重建并重启 `dataset-service`/`model-service` 镜像。
- **未做（非必需）**：B 重量方案（任务表 + worker 持久化）；如需进程重启可恢复的校验，可后续迁移到 worker 模式。
