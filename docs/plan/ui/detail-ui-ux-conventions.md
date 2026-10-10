# 详情页 UI/UX 规范化 —— 可复用提示词（范本：基座模型详情）

> 用途：将「基座模型详情页」的 UI/UX 设计提炼为可复用提示词，推广到其它详情页（如数据集详情页）。
> 关联：`docs/plan/ui/list-ui-ux-conventions.md`（列表页）、`docs/plan/ui/detail-tab-unresponsive-analysis.md`（性能约定）
> 使用方式：把下文「目标页」替换为要改造的详情页；标注【替换】处按该页领域替换，其余为通用约定。

---

## 0. 设计原则（一句话）
**状态可见、层级清晰、动作就近、渐进披露、语义色统一、性能友好（懒加载 / 分批 / 静默刷新）。**

## 1. 页面骨架（Shell）
- **PageHeader**：标题 + 副标题（描述）+ 面包屑（列表→详情）+ 右侧操作区：
  - 上/下条翻页 `RecordNav`（沿用列表过滤/分页上下文）
  - **主操作**（主色按钮）、**危险操作**（危险色描边按钮）
- **顶部概览条 / 状态卡**：`StatusTag` + 关键指标若干 + 右侧**结论文案**；左边框 3px 语义色条（`tone-*`），结论文案随 tone 变色。
- **DetailTabs**（见 §2）：分段胶囊、可吸顶（`sticky`）、图标 + 文案 + **徽标计数**。
- **Tab 内容懒加载**：仅首次进入该 Tab 才拉数据；切换 Tab **不得整页重挂载**（见 §10）。
- 对话框/确认框集中在 Shell 挂载。

## 2. Tab 导航 `DetailTabs`（通用组件）
- 分段胶囊：浅底容器 + 激活项实底抬升 + 主色文字 + `shadow-sm`。
- 语义/无障碍：`role=tablist`/`role=tab`、`aria-selected`、`aria-controls`、`tabindex`、左右方向键循环。
- 可选吸顶：`position: sticky; top:0; z-index:20;` 背景 `--bg-page`。
- 徽标：待处理计数（默认 warning 底，激活态主色）；`0/undefined` 不显示。
- 激活项滚入：**仅滚动 Tab 轨道**（不 `scrollIntoView` 整页）。

## 3. 概览 Tab 布局（通用）
- **2 列响应式 grid**（`≤900px` 单列），每格为 `panel`：
  - `panel` = `--bg-surface` + 1px `--border` + `--radius-lg` + `--space-5`。
  - 强调 panel：`border-inline-start: 3px` 语义色（配 `tone-*`），可加浅色底（`color-mix`）。
- **键值列表 `kv`**：`dt` 次级/小字、`dd` 主色；`dashed` 底分隔；`kv--rows` 单列 + 长值 `mono break` + `CopyButton`。
- **折叠 panel**：`panel__toggle`（caret 旋转 + 计数）+ 内容 `DataTable`。
- 次级指标用 `StatusChip`（如 量化/可商用）；敏感/专业字段配 `InfoHint` 说明。

## 4. 状态与语义色（统一）
- 统一设计令牌：`success / warning / danger / info / primary / neutral / accent`（violet 为 `accent`，用于「结构/组件」类语义）。
- 展示组件：`StatusTag`（领域状态）、`StatusChip`（`soft` 默认 / `solid` 强调 / `plain` 仅彩字）。
- **色调映射集中**在 `utils/assetStatus.ts` 之类工具（函数式 `xxxTone()`）：任务级与条目级**共用同一套色调**，避免「任务失败但条目可用」的歧义。
- 结论文案、左边框、徽标色调与状态枚举保持一致。

## 5. 动作（就近 + 语义分组）
- 语义样式：`详情/次操作` = `link--muted`（中性）；`主操作` = `link--primary`；`危险` = `link--danger`。
- 分组顺序：`中性 ‖ 主操作 ｜ 次操作 ‖ 危险`，组间用 `actions-sep`（1px 竖线）。
- 图标 14px，文案 `--fs-small`；禁用用 `disabled`；主页面级操作放 PageHeader（主色/危险按钮）。
- 无冗余 `⋯` 下拉（动作少时平铺；动作多可保留 `DropdownMenu` 收纳）。

## 6. 进度与实时
- 进度组件：`PullProgress`（进度条 + 速率 + ETA + 文件数 + 复用字节 + 自动重试提示）。
- 进度条：列表态 **4px**；紧凑态 **3px**、区块居中、文字与条左端对齐、`nowrap` + 气泡。
- 实时策略：**SSE 为主 + 轮询兜底**（SSE 连通即停轮询；**无 SSE 事件的异步任务必须显式轮询**）。
- **静默刷新不置骨架**；页面隐藏暂停轮询。

## 7. 条目列表类 Tab（大数据集交互）
- 工具条：状态筛选 **chips**（dot + 计数）、搜索、排序（键 + 升降序）、**密度**（详细/紧凑）、**视图**（卡片/列表）。
- 属性用 `StatusChip` + `InfoHint`；「需处理置顶」稳定分区（不改组内顺序）。
- 分组（如组件）可折叠：`group-toggle`（caret + 名称 + 计数），子项缩进。
- **批量操作条**：选中计数 + 批量动作（放行/重扫/禁用/对比/取消），危险项标红。
- **分批渲染**：首屏 N（如 60），「加载更多」+120；筛选/排序变化重置。
- 可选：图例、导出 CSV。

## 8. 文件预览
- 可折叠 `FileBrowser`（`fetch-tree` + `loader`/`url-loader`/`archive-loader`/`parquet-loader`）。
- 进行中任务开启自动刷新；预览开启/隐藏时暂停。

## 9. 空 / 错 / 加载态（三态齐全）
- `SkeletonBlock`（加载）/ `ErrorState`（retry）/ `EmptyState`（含引导动作）。
- 加载骨架**不得阻塞 Tab 切换**或整页交互。

## 10. 性能与体验约定（务必遵守）
- **query 只承载可分享/可恢复状态**；详情 Tab 状态若写入 query，**不得因此整页重挂载**（路由 `meta.queryKey` 白名单，仅列表页开启）。
- 异步任务（校验/导入等）**无 SSE 时显式轮询**；动作完成后**重排刷新定时器**（保证快刷，如 3s 内更新）。
- 慢/关键接口按请求放宽超时；轮询**不得叠加**（完成后再排 / 在途守卫）。
- 组件卸载清理定时器；页面隐藏暂停。

## 11. 可访问性与 i18n
- `aria-*`、`focus-visible` 聚焦环、键盘可达（Tab 方向键、`RecordNav` 上下条）。
- 全部文案 `tt('key', '默认中文')`；18 语种同步（`npm run i18n:audit` = 0 缺失）。

## 12. 复用清单：【替换】项
1. 状态枚举与 tone 映射函数（`utils/xxxStatus.ts`）。
2. 概览条 / 基本信息**指标集**。
3. **Tab 集合**及各自面板。
4. **动作集**与语义分组。
5. **实时数据源**（SSE topic / 轮询条件 / 可见进度）。

## 13. 验收清单
- 视觉：分段胶囊吸顶、语义色一致、panel/kv/折叠风格统一。
- 交互：Tab 切换**无整页骨架/无重挂载**、懒加载、折叠记忆、滚动位置恢复。
- 实时：SSE + 兜底、快刷、静默刷新无骨架。
- 三态齐全；i18n 全绿；a11y；无 `pageerror`。
- 截图留存（浅/深）。

---

## 附：目标页适配示例 —— 数据集详情页（下一步推广）
| 维度 | 基座模型详情（范本） | 数据集详情（目标） |
|---|---|---|
| 概览条指标 | 状态/大小/权重格式/更新时间 | 状态/数据类型/大小/版本数/更新时间 |
| Tab 集合 | 概览 / 扫描分析 / 模型变体 | 概览 / 扫描分析 / 数据子集 / 派生版本 |
| 状态映射 | ready/failed/…（assetStatus） | created/uploading/validating/available/invalid/archived → registered/syncing/ready/failed/archived |
| 动作集 | 拉取/重拉/暂停/取消；变体放行/重扫/禁用 | 导入/追加文件/校验/新建版本/删除；导入暂停/恢复/取消 |
| 实时源 | `pull-task`、`asset-scan` SSE；拉取轮询兜底 | `dataset-import` SSE；**校验无 SSE → 显式轮询 + 动作后重排快刷** |
| 预览 | 基模文件树 | 数据集文件树（含 Parquet/压缩包） |
| 条目列表 | 变体（chips/list、密度、分组、批量） | 版本列表（表格）+ 子集/派生（面板） |

---

## 已推广记录
| 目标页 | 提交 | 说明 |
|---|---|---|
| 数据集详情页 | `f2aac8c5` | 概览条（加粗指标 + 语义色结论）、Tab 吸顶、SkeletonBlock、StatusChip、最近导入面板 tone 左边框 + 动作图标语义分组；浅/深 Playwright 校验通过 |

> 待推广：训练详情、推理详情、评测详情、压缩详情、镜像类详情（基础镜像/规约/构建/产物）。
