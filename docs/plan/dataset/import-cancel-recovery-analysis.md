# 数据集导入「后台取消 / 无恢复入口」—— 同类问题分析与修复

> 状态：**已修复** — 代码仓提交 `fa7140af`（Gitee origin）
> 背景：基模拉取存在「看门狗误杀 + 重派伪取消 + 后台产生 CANCELLED + 无恢复入口」，排查数据集导入是否同类。

## 一、对照结论（同类）
| 维度 | 拉取 | 数据集导入（修复前） | 同类 |
|---|---|---|---|
| 后台置 CANCELLED | 已改 FAILED | `dataset TaskResultHandler`：importer cancelled 上报 → `ImportEvent.CANCEL`(system) | 是 |
| 重派伪取消 | puller 加 `_superseded` | importer `_restart` 停旧启新，旧执行仍上报 cancelled | 是 |
| 看门狗总时长误杀 | 改 updated_at 无进展 | `DatasetImportWatchdog` 用 `started_at`（默认 30min） | 是 |
| 重试耗尽 | FAILED→ABANDONED | 同（`DatasetImportAutoRetryJob`） | 一致，合理 |
| 重复任务 supersede | 改 FAILED→ABANDONED | 已是 `abandonFailed`(ABANDONED) | 无问题 |
| 取消后恢复入口 | 加「重新运行」 | `canRetryImport` 不含 cancelled | 是（较弱） |
| 收尾无进度 | 加 verifying/uploading 保活 | 流式上传有进度、末尾仅写 manifest | 较轻 |

## 二、修复（A+B+C+D）
- **A 后台不产生 CANCELLED**：`dataset TaskResultHandler` 对 importer 非用户 `cancelled` 上报 → 置 **FAILED**（“importer stopped unexpectedly”，可重试）。
- **B 抑制重派伪取消**：`dataset-importer` `ImportTaskHandler` 新增 `_superseded`；`_restart` 停旧启新时旧执行不再上报 `cancelled`（与 model-puller 一致）。
- **C 看门狗改无进展超时**：`DatasetImportWatchdog` 改用 `updated_at`（新增 `DatasetImportTaskRepository.listRunningStaleBefore`），长导入持续上报进度不误杀。
- **D 恢复入口**：`datasetLifecycle.canRetryImport` 覆盖 `cancelled`；前端「失败→重试、取消/放弃→重新运行」，`DatasetsView`/`DatasetDetailView` 统一入口（重新运行 = 新建导入任务、复用 staging）。

## 三、验证
- `typecheck`/`i18n`/`build`；dataset-service **31** 用例通过。
- dataset-importer 镜像重建（含 `import` 自检）通过；容器内校验 `_superseded` 存在于 `_execute`/`_restart`。
- 数据集详情/列表 Playwright 回归全绿（无 pageerror）。

## 四、语义总结（与拉取一致）
- 用户取消 → `CANCELLED`（终态，可「重新运行」新任务恢复）
- 看门狗无进展 / 下载失败 / importer 异常停止 → `FAILED`（可自动/手动重试同一任务）
- 重试耗尽或被取代 → `ABANDONED`（终态，可「重新运行」新任务恢复）
- 后台进程不产生 `CANCELLED`（仅用户主动）。
