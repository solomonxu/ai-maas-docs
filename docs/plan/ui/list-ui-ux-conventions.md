# 列表页 UI/UX 规范化：可复用提示词

> 用途：把「基座模型列表」已落地的 UI/UX 规范，作为**提示词**复用，快速推广到其它列表页（如数据集列表）。
> 用法：复制下方提示词，替换 `{{占位符}}` 后交给编码助手执行。

---

## 提示词正文

```text
# 任务：将「列表页 UI/UX 规范化」推广到目标列表页

## 目标
参照「基座模型列表」已落地的 UI/UX 规范，对目标列表页 {{LIST_PAGE}}（如 maas-web/src/views/datasets/DatasetsView.vue）做一致化重构：列设计与对齐、行操作语义化、工具栏分组、筛选端到端有效、语义色、字号层级、响应式与深浅色一致。只做 UI/UX 与必要的前后端筛选打通，不改业务语义。

## 适用范围与占位符
- 目标列表页：{{LIST_PAGE}}
- 列表数据接口：{{LIST_API}}（前端 {{API_FILE}}，后端 {{CONTROLLER}}）
- 详情路由：{{DETAIL_ROUTE}}
- 首列主链接字段：{{NAME_FIELD}}；副行字段：{{SUB_FIELDS}}
- 行状态字段/状态标签类型：{{STATUS_FIELD}} / StatusTag type="{{STATUS_TYPE}}"
- 行筛选字段（需端到端）：keyword、{{FILTER_FIELDS}}（如 status、source、family）

## 硬约束（务必遵守）
- Vue 3 <script setup> + TypeScript + Pinia + Vue Router + Axios + Vue I18n；禁止引入任何第三方 UI 库，复用自研 @/components/common/*。
- 图标用 @phosphor-icons/vue。
- i18n 18 语种全同步，npm run i18n:audit 必须 0 缺/多/空；文案用 tt('key','中文默认') 兜底。
- 复用设计令牌（styles/tokens.css）：颜色/间距/圆角/字号一律用 var(--...)，不硬编码；浅/深色都需正确。
- 金额/日期等用现有 useLocale() 的 formatBytes/formatDate/formatRate/formatDuration。

## 复用组件与工具（优先复用，不新造）
- 表格：components/common/DataTable.vue（已支持：align、minWidth 横向滚动、固定列毛玻璃、分页/排序/选择）。
- 语义标签：components/common/StatusChip.vue（浅底+图标）；StatusTag.vue（既有状态映射）。
- 进度：components/common/PullProgress.vue / ProgressBar.vue（紧凑态 3px、居中、文字与条左端对齐）。
- 语义色调映射：utils/assetStatus.ts（variantStatusTone/dispositionTone/securityTone/roleTone/schedulableTone/pullTaskTone）。
- 详情骨架/Tab（若涉及详情）：common/DetailTabs.vue、PaginationBar.vue、RefreshControl.vue、usePageSize/useAutoRefresh。

## 列设计规则
1. 合并低价值列：把「系列/档位」等并入主列的副行（cell-main + cell-sub，副行淡色）。列数尽量精简。
2. 来源类列：仅展示图标/短徽标（如 HF/MS），具体仓库/ID 放主列副行，淡色。
3. 宽度：为每列设合适 width；总宽不足时允许 DataTable 横向滚动（不压缩截断）。
4. 对齐：状态、时间、进度等居中（表头与内容一致）；数值（大小）右对齐 + tabular-nums。
5. 字号层级：次级列（如更新时间）用 --fs-small（12px）+ 次级色；主列/常规列 13px（相对 body 收细），避免“同时偏大”。
6. 进度列：加宽以完整显示；进度条细（3px）、居中、其下文字与进度条左端对齐；white-space:nowrap + 溢出省略，整格加 title 气泡显示全文。
7. 行内不折行：主列变单行（如副行内联），使行高稳定、操作列占满。

## 行操作规则（语义化 + 分组 + 固定宽）
- 顺序与语义：详情（中性，PhEye） ‖ 分隔符 ‖ 主操作（主色，如 拉取/重拉/重试/开始） ｜ 次操作（中性，如 暂停/恢复/分析） ‖ 分隔符 ‖ 危险（取消/删除，红）。
- 颜色：主=--color-primary；中性=次级；危险=--color-danger；正向（如放行）=绿色（btn--success/success 色）。
- 形态：保留平铺（不折叠），操作列固定宽度并右对齐、nowrap；文字 12px、图标 13px；末尾单元格 padding-inline-end: space-2。
- 复用全局 styles/controls.css 的 .link/.link--primary/.link--muted/.link--danger/.btn/.btn--success/.btn--sm。
- 「全选」需为可切换（全选 ⇄ 取消全选，带激活态）。

## 工具栏/筛选规则
- 工具条分组（视图切换 ｜ 搜索/排序 ｜ 批量/工具/导出），用分隔符 + 图标；排序方向用图标按钮（PhSortAscending/Descending）+ tooltip。
- 筛选必须端到端有效：若后端未支持，则补齐 —
  - 前端：列表请求带上 keyword 与 {{FILTER_FIELDS}}；
  - 后端：查询 DTO/领域值对象/仓储 where 增加对应条件（keyword 支持 ID 精确 + 名称/仓库模糊；筛选精确匹配，大小写归一）；
  - 验证：接口用不同参数返回不同结果集，UI 选择后请求含该参数且行数变化。

## 语义色彩与状态
- 状态/处置/安全级别/角色/可调度 等，统一用 StatusChip（浅底彩字）或既有 StatusTag；避免同色一列（如三个“危险”类用 红/深灰/紫 区分）。
- 汇总/主要状态卡可用左语义色条（border-inline-start + tone）。
- 颜色语义：成功=绿、进行中/主=蓝、待确认/暂停=琥珀、失败/隔离=红、中性/取消=灰、组件=紫（accent）。

## 可访问性与响应式
- 键盘：focus-visible 聚焦环；aria-*（tablist/tab、排序按钮、开关）。
- 窄屏：表格横向滚动；工具栏换行；关键列保留、低价值列可隐藏。

## 交付与验证（必须）
1. 逐项列出改动文件与说明（列合并/对齐/字号/操作分组/筛选打通等）。
2. npm run typecheck、容器内 npm run build、npm run i18n:audit（18 语种）全通过。
3. 若改后端：mvn -o -pl {{MODULE}} -am compile -DskipTests 通过；重建对应服务镜像并重启（如需）。
4. Playwright 冒烟（截图 + 断言）：主列/副行、状态与时间居中、操作右对齐不折行、筛选下拉触发请求且结果变化、无 pageerror。
5. 输出改动清单模板：
   [列] 合并 X→Y；来源仅图标；进度列加宽/细化/对齐；字号 13/12px
   [操作] 语义分组+图标+右对齐固定宽(12px/icon13px)
   [筛选] keyword/source 端到端（前端传参 + 后端仓储过滤）
   [组件] 复用 StatusChip/DataTable(minWidth 横向滚动, align, 毛玻璃固定列)
   [验证] typecheck/build/i18n:audit + Playwright ALL PASS

## 不要做
- 不引入第三方 UI 库/图标库；不改业务语义与接口契约（仅按需扩展筛选参数）；不做与目标页无关的全局重构。
```

---

## 用法示例
- `{{LIST_PAGE}}` → `maas-web/src/views/datasets/DatasetsView.vue`
- `{{LIST_API}}` → 数据集列表接口（前端 `maas-web/src/api/datasets.ts`，后端 `DatasetController`）
- `{{FILTER_FIELDS}}` → `status`、`source`、`format`
- `{{STATUS_TYPE}}` → `dataset`（取决于 StatusTag 映射）

## 参考实现（基座模型列表）
- 页面：`maas-web/src/views/base-models/BaseModelsView.vue`
- 共享：`components/common/{DataTable,StatusChip,PullProgress}.vue`、`utils/assetStatus.ts`、`styles/controls.css`
- 端到端筛选示例：base-model 列表的 `source` 过滤（前端传参 + 后端仓储 `where`）
