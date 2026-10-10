# 大模型拉取「99% 卡住 → 已取消」—— 根因分析与修复

> 状态：**已修复** — 代码仓提交 `58937336`（Gitee origin）
> 现象：拉取较大基模（ZhengmingYu/DMAD text-to-video，57GB），前段 ~50MB/s 很快，末尾在 99% 徘徊不前，最终显示**已取消**（而 puller 日志显示 succeeded）。

## 一、根因（由任务事件日志坐实，task 2108967780172754945）
事件序列：
- 01:08:52 running（resume 后）
- **01:39:12 `fail` running→failed，operator=system，detail=`watchdog_timeout`** —— 恰为 running 起算 **~30 分钟**
- 01:39:40 `retry` failed→pending（operator=user，retryCount=1）
- 01:39:40.8 `cancel` pending→cancelled（operator=system）
- puller 日志：01:44 仍在报 99.9%，**01:47:56 `TASK COMPLETE status=succeeded`**

两处缺陷叠加：
1. **看门狗按“总时长”而非“进展”判定**：`ModelPullWatchdog`（`maas.pull.watchdog-ttl-ms` 默认 30min）用 `started_at` 判定，
   57GB 模型正常也会 >30min → 被误判 `pull timeout` 置 failed（且未真正停 puller）。
2. **重试重派触发伪取消**：用户点“重试” → `retry()` 重新下发 `TASK_ASSIGN`（同 taskId）→ puller `on_assign` 见已注册 →
   `_restart` 对旧执行 `cancel_event.set()` → 旧执行 `_execute` 走 `CancelledError` 分支并**上报 `cancelled`** →
   后端把刚重试成 pending 的任务置 **cancelled**；随后新执行恢复下载并成功，但任务已终态 → **成功被忽略** → UI 停在 99%。

## 二、修复（A+B+C）
- **A 看门狗改“无进展超时”**：`ModelPullWatchdog` 改用 `updated_at`（每次进度持久化刷新）判定，
  新增 `ModelPullTaskRepository.listRunningStaleBefore`；长任务只要持续上报进度即不误杀（默认 30min 无进展才失败）。
- **B 抑制重派伪取消**：puller `handler` 新增 `_superseded`；`_restart` 停旧启新时旧执行不再上报 `cancelled`
  （与“暂停”一样抑制终态），避免误置刚重试的任务为 cancelled。
- **C 收尾阶段保活**：`downloader` 在校验（sha256 全量读盘）与上传阶段切 `phase`(verifying/uploading) 并每 15s 强制上报，
  UI 不再假死在 99%，同时保持 `updated_at` 新鲜配合 A。

## 三、验证
- base-model-service **32** 用例通过。
- puller 镜像重建（含构建期 `import model_puller` 自检）通过；容器内校验 `_activity` 周期上报（0.18s/0.05s=3 次）
  与 `_superseded` 存在于 `_execute`/`_restart`。
- 运行时对已拉取模型重拉：即时 `succeeded 100% retry=0`，无伪取消。

## 四、仍可优化（后续）
- 真正的暂停/取消需确认 puller 立即停止（watchdog 失败时未下发 cancel，本次放大了浪费）。
- 校验/上传若极大文件耗时 > 看门狗 TTL，仍依赖 C 的保活；可考虑按“阶段+活动”综合判定。
