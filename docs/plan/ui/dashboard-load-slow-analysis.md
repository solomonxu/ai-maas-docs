# 仪表盘首屏 15s 无数据 —— 根因分析与修复

> 状态：**已修复** — 代码仓提交 `d8e2e9ff`（Gitee origin）
> 现象：进入/刷新仪表盘，先只有框架无数据，约 **15s** 后数据才出现。

## 一、根因
`DashboardView.load()` → `getDashboardStats()` 用 `Promise.all` 等 7 个请求全部 settle 才渲染。
实测各接口耗时：

| 接口 | 耗时 |
|---|---|
| 其余 6 个（base-models/datasets/training-jobs/pull-tasks/nodes/components…） | < 20ms |
| **`GET /stats/storage`** | **16.12s** |

`/stats/storage` 超过全局 `axios timeout=15000ms`，15s 超时后 `.catch(()=>null)` 才 settle，
`Promise.all` 因此阻塞约 15s → 整页数据 15s 后才出现（时间点与 15s 完全吻合）。

**后端 `StatsController.storage()`（dataset-service）**：对 4 个桶 `listObjects(recursive=true)`
**全量遍历**累加体积/对象数（`maas-models` 对象多，主耗时），且每次请求重算无缓存。

## 二、修复
**前端（解耦）**
- `getDashboardStats()` 移除 `/stats/storage`；新增 `getStorageStats()` **独立加载**，存储卡先显「计算中…」再回填。
- `DashboardView` 去掉 `load()` 中重复且未使用的 `listAllNodes()`。

**后端（根治）**
- `StatsController` 不再 `listObjects`：改抓 MinIO 集群指标（`minio_cluster_usage_total_bytes`、
  `minio_cluster_capacity_usable_free_bytes/total_bytes`）一次解析，**30s TTL 缓存**；响应保留
  `usedBytes/freeBytes/totalBytes`（`buckets` 置空，前端未使用）。

## 三、验证
- `/stats/storage`：16.12s → **41ms（冷）/ ~3ms（缓存）**。
- 仪表盘首屏主数据 **401ms** 渲染（原 ~15s）；存储卡随后独立回填（示例 511.2 GB，空闲 117.0 GB / 共 800.0 GB）。
- Playwright：首屏 <2s、存储卡回填、独立调用、无 `pageerror`；dataset-service 31 用例通过。

## 四、通用原则
仪表盘聚合**不应被单个慢/可选接口拖累**：慢数据用「独立加载 + 占位」，避免 `Promise.all` 全量等待；
后端聚合类接口避免全量遍历对象存储，优先用指标/元数据 + 缓存。

## 五、扩展排查（全项目「全量遍历/无缓存」隐患）与修复
> 提交 `cec29a1f`（Gitee origin）

| # | 位置 | 类型 | 处置 |
|---|---|---|---|
| 1 | `dataset-service StatsController.storage` | MinIO 全量递归遍历 4 桶（16.12s） | 已修（`d8e2e9ff`） |
| 2 | 前端 `TrainingView`/`EvalView`/`ArtifactsView` 名称映射 | 每次列表刷新重复拉 200+200(+200) 行，无缓存 | **已修**：新增 `utils/lookupCache`（60s TTL + single-flight），改 `cachedLookup`；验证训练列表 size=200 请求各 1 次 |
| 4 | 拉取任务详情实时文件列举 `listBaseModelFiles` | 轮询中每次全量递归列举 MinIO | **已修**：节流 10s |
| 5 | base-model `SourceMetaClient.query/revisions` | 请求内直连 HF/MS，无缓存 | **已修**：TTL 缓存（成功 10min/失败 30s）；验证 source-meta 1.5s→4ms、source-refs 0.32s→4ms |
| 3 | node-service `/nodes`、`/components` | 内存 `listAll` + 过滤/分页 | 低：已有短 TTL 缓存；数量增长后改 DB 级分页 |
| 6 | admin `/admin/usage`、`/admin/overview` | 日期范围查询 / `count(*)` | 低：有界，无需处理 |

**已做对的**：node-service `listAll` TTL 缓存 + 批量查询避免 N+1、列表装配不触发 registry 查询；
eval/artifact 列表已 `selectPage` 分页；estimate 已 TTL 缓存；文件树浏览已用非递归 `listLevel` 懒加载。

