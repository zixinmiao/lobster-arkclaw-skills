---
name: lobster-fitting-bitable-sync
description: 将试衣合并结果映射成适合飞书多维表格写入的结构。适用于需要把“单客单商品单次试衣”记录 upsert 到飞书多维表格台账、补齐会员字段、保留 session 归属与素材映射、生成写入摘要时使用。
---

# 试衣飞书多维表格沉淀

把结构化试衣数据整理成适合飞书多维表格写入的结果。

## 依赖
- 当任务涉及飞书多维表格字段设计、记录写入、视图或公式语义时，优先配合 `lark-base` 类能力执行。
- 若当前缺少 `bitable_binding`、Base 或目标表结构，先使用 `lobster-fitting-bitable-bootstrap` 补齐基础设施，再生成正式写表载荷。
- 当前项目的共享表配置放在 `../configs/feishu-bases.json`。执行写表前，先读取对应 `baseToken / tableId / fields / dedupeKeys`，不要在 skill 内手写 token。

## 输入
- `input_bundle`
- `session_link`
- `fitting_records`
- `media_records`
- `write_decision`
- `bitable_binding`
- `project_config`（可选；若缺失，默认读取 `../configs/feishu-bases.json`）
- `instance_context`（可选；用于补 `store_name / source_agent_id` 等来源字段）

## 当前共享表绑定
以 `../configs/feishu-bases.json` 为准。

### 主表
- Base：`导购小龙虾试衣台账`
- `baseToken`: `JwYkbN8epa6466sJjzacBz9VnYe`
- `tableId`: `tbl4yCJIoezUR0hD`
- `viewId`: `vew1wtcott`

## 数据粒度
### 主表粒度
主表 `试衣商品记录` 的一条记录定义为：

> 一件商品 + 一个客人 + 一个 session = 一条记录

若同一 session 中试了 3 件商品，应写入 3 条主记录。

## 字段映射
### fitting_record -> 当前试衣主表真实字段
按 `../configs/feishu-bases.json -> bases.tryon.fields` 输出，当前线上字段为：
- `导购小龙虾试衣台账`（用作 `record_id` 主键字段）
- `customer_id`
- `member_mobile_last4`
- `liked_points`
- `session_id`
- `session_status`
- `source_channel`
- `guide_name`
- `source_sender_id`
- `fit_feedback`
- `try_on_result`
- `source_bundle_id`
- `ocr_confidence`
- `raw_notes`
- `size`
- `body_effect_desc`
- `source_chat_id`
- `tag_price`
- `not_buy_reason`
- `operator_id`
- `product_name`
- `is_member`
- `source_message_ids`
- `color`
- `followup_intent`
- `store_name`
- `member_identity_status`
- `disliked_points`
- `is_high_intent`
- `fitting_time`
- `sku`
- `operator_name`
- `fitting_date`
- `product_code`

### 来源字段兜底规则
如果实例侧还没有门店配置，不要阻塞写表：
- `store_name` 默认写 `unknown`
- `source_channel` 默认写 `openclaw`
- 若链路里能拿到实例标识，优先写入 `operator_id / operator_name / source_sender_id`
- 当前表里还没有独立 `source_agent_id / source_store_id` 字段时，不强依赖；先依靠 `operator_id / source_chat_id / source_sender_id / session_id` 做追踪

### 可扩展关联表
以下表由其他 skill 负责生成或更新，但应与试衣主表保留可关联字段：
- `线索回访表`（由 `lobster-followup-lead-sync` 负责）
- `会员画像表`（由 `lobster-member-profile-sync` 负责）

### 商品主值映射约束
- 多维表格中的商品主字段，应直接映射自当前 `fitting_record.product_name / product_code / sku`。
- 当本次输入包含新的吊牌 OCR 结果时，写表层不得再回退使用历史上下文商品名覆盖当前主值。
- 若上游 `merge_notes` 已标记商品冲突，写表层应优先保留本次 bundle 的商品主值，并把冲突说明保留到摘要字段，而不是把旧商品直接写进主表。

## dedupe 建议
### 主表 dedupe_key
优先使用 `../configs/feishu-bases.json -> bases.tryon.dedupeKeys`：
- `record_id`

若上游未显式生成 `record_id`，退化为：
- `session_id + customer_id + product_code`

## 输出目标
输出 `sync_result`：
- `tables`: 各表待写入记录
- `write_mode`: `upsert`
- `dedupe_keys`
- `sync_status`
- `sync_summary`

## 执行规则
- 缺 binding 时，优先交给 `lobster-fitting-bitable-bootstrap` 补齐 binding；若当前链路无建表能力，再输出待写入载荷，不伪造写入成功。
- 若已有 `binding_scope = project` 的可用 binding，则后续导购默认复用同一套 Base，不重复建表。
- 缺关键字段时，标记 `partial` 并在 `sync_summary` 说明。
- 若只拿到 `session` 级摘要、尚未拆成 `fitting_records`，应优先提示上游先完成记录粒度转换，不要擅自按旧结构落表。
- 若 `session_link.link_action = pending_confirm`，允许输出待写入载荷，但应在 `sync_status` 或 `sync_summary` 中明确说明当前归属仍待确认。
- 若 `write_decision.allow_write = false`，则连接器/写入器必须停止向主表写入；其中当 `write_decision.write_block_reason = PENDING_MEDIA_NOT_READY_TO_WRITE` 时，应直接跳过正式记录创建。
- 若 `input_bundle.bundle_status = pending_media` 且 `fitting_records = []`，则不得向主表 `试衣商品记录` 写入任何正式记录；此时仅允许输出待处理信息。
- 若上游只提供单张图片且无明确文本，不得在 `sync_summary` 或任何落表字段中推断“成交/已购买/已完成”。
- 输出应便于后续由连接器、脚本或多维表格写入器直接执行。

## 输出格式
返回：
```json
{"sync_result": {
  "tables": {
    "试衣商品记录": []
  },
  "write_mode": "upsert",
  "dedupe_keys": {
    "试衣商品记录": ["record_id"]
  },
  "sync_status": "ready | partial | pending_confirm | no_binding",
  "sync_summary": "..."
}}
```
