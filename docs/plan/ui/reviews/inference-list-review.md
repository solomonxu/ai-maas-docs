# 推理服务列表「有数据时」肉眼复核清单

> 页面：``/inference` → `maas-web/src/views/inference/InferenceView.vue``
> 关联规范：`docs/plan/ui/list-ui-ux-conventions.md`
> 复现：≥2 个推理实例，覆盖 `running/stopped/failed/cold`；至少 1 个有端点地址。
> 通用项：A/E/F/G 同 `compress-list-review.md`（见文末节摘要）。

## B. 列与对齐（本次改动重点）
| 列 | 预期 |
|---|---|
| 名称 | 左对齐；可点击跳详情；13px |
| 基座模型 / LoRA | 左/常规；无值 `—` |
| 后端 | **居中**；`StatusTag(engine)` 或文本 |
| 状态 | **居中** + `StatusTag(inference)`（含 `cold`） |
| 端点地址 | 等宽（mono）；过长省略/气泡 |
| GPU | **右对齐** tabular-nums |
| 创建时间 | **居中**、12px、次级色、`YYYY-MM-DD HH:MM:SS`（无 T/毫秒/时区） |
| 操作 | **右对齐**；文字 12px、图标 13px |


## C. 操作语义化
- [ ] `详情`(中性 `PhEye`) ‖ `对话/启动`(主色 `PhChatCircleDots/PhPlay`) ｜ `唤醒/释放显存/编辑`(中性 `PhLightning/PhArrowLineDown/PhPencil`) ‖ `停止/删除`(危险 `PhStop/PhTrash`)。
- [ ] **已去除冗余 `⋯` 下拉**（动作平铺）。

## D. 工具条
- [ ] `搜索 ‖ 分隔符 ‖ 引擎筛选 ‖ 状态筛选 ‖ 重置(图标)`。
- [ ] **端到端**：`engine`/`status` 生效。

## A. 数据与基础
- [ ] 列表渲染行；分页正确；加载骨架 / 错误态+重试 / 空态（含入口）。
- [ ] 筛选/搜索后端生效：操作后行数变化，Network 请求含对应查询参数（端到端）。
- [ ] 「重置」清空筛选并回到第 1 页。

## E. 视觉/一致性
- [ ] 全表内容 13px；表头与内容对齐一致。
- [ ] 浅色/深色正常；窄屏可横向滚动、不压缩截断。
- [ ] 数值列使用等宽数字（tabular-nums）。
- [ ] 18 语种切换后表头/按钮/状态本地化（无缺失/英文占位）。

## F. 交互细节
- [ ] 行内动作不误触其它按钮（`@click.stop`）。
- [ ] 运行中任务随 SSE/轮询更新；终态停止高频刷新。
- [ ] `focus-visible` 聚焦环；无 `pageerror`。

## G. 与规范文档一致
- [ ] 对照 `docs/plan/ui/list-ui-ux-conventions.md`「统一实现约定」逐项通过。
- [ ] 截图留存（浅/深各一张）。

---

## 复用到其它列表页
- B（列与对齐）：按目标页替换列与预期；通用——状态/时间/进度居中、数值右对齐、时间 `YYYY-MM-DD HH:MM:SS`。
- C（操作语义化）：`详情`=中性 `PhEye`、主操作=主色、危险=红 `PhTrash/PhX`、无冗余下拉。
- D（工具条）：`搜索 ‖ 分隔符 ‖ 筛选 ‖ 重置(图标)`。
