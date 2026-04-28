---
name: lobster-fitting-bitable-sync
description: 将试衣合并结果映射成适合飞书多维表格写入的结构。适用于需要把“单客单商品单次试衣”记录 upsert 到飞书多维表格台账、补齐会员字段、保留 session 归属与素材映射、生成写入摘要时使用。
---

# 试衣飞书多维表格沉淀

把结构化试衣数据整理成适合飞书多维表格写入的结果。

## 依赖
当任务涉及飞书多维表格字段设计、记录写入、视图或公式语义时，优先配合 `lark-base` 类能力执行。
若当前缺少 `bitable_binding`、Base 或目标表结构，先使用 `lobster-fitting-bitable-bootstrap` 补齐基础设施，再生成正式写表载荷。

## 输入
- `input_bundle`
- `session_link`
- `fitting_records`
- `media_records`
- `write_decision`
- `bitable_binding`

## 目标表
### 主表
- `试衣商品记录`

### 辅助表
- `试衣素材索引`
- `试衣Session索引`（可选，用于追踪 session 生命周期和上下文归属）

### 可扩展关联表
以下表建议由其他 skill 负责生成或更新，但应与试衣主表保留可关联字段：
- `线索回访表`（由 `lobster-followup-lead-sync` 负责）
- `会员画像表`（由 `lobster-member-profile-sync` 负责）

## 数据粒度
### 主表粒度
主表 `试衣商品记录` 的一条记录定义为：

> 一件商品 + 一个客人 + 一个 session = 一条记录

也就是“单客单商品单次试衣”粒度。

若同一 session 中试了 3 件商品，应写入 3 条主记录。

## 字段映射
### fitting_record -> 试衣商品记录
- `record_id`
- `session_id`
- `session_status`
- `fitting_date`
- `fitting_time`
- `guide_name`
- `store_name`
- `operator_id`
- `operator_name`
- `source_chat_id`
- `source_sender_id`
- `customer_id`
- `is_member`
- `member_mobile_last4`
- `member_identity_status`
- `sku`
- `product_code`
- `product_name`
- `color`
- `size`
- `tag_price`
- `ocr_confidence`
- `try_on_result`
- `body_effect_desc`
- `fit_feedback`
- `liked_points`
- `disliked_points`
- `not_buy_reason`
- `followup_intent`
- `is_high_intent`
- `source_channel`
- `source_bundle_id`
- `source_message_ids`
- `raw_notes`

### 现有飞书试衣表可增量补充字段
若当前线上飞书试衣表仍较轻量（如仅有日期、导购、会员尾号、试穿单品、客户反馈、结果、客户体型描述），建议增量补充以下字段：
- `是否生成回访线索`
- `回访原因`
- `建议触达时机`
- `触达触发条件`
- `建议回访内容`
- `线索优先级`
- `是否建议更新会员画像`
- `会员画像补充摘要`

### media_record -> 试衣素材索引
- `media_id`
- `record_id`
- `session_id`
- `product_code`
- `media_type`
- `media_url`
- `media_source`
- `message_id`
- `captured_at`

### 商品主值映射约束
- 多维表格中的“试穿单品”或等价商品主字段，应直接映射自当前 `fitting_record.product_name` / `product_code` / `sku`。
- **当本次输入包含新的吊牌 OCR 结果时，写表层不得再回退使用历史上下文商品名覆盖当前主值。**
- **若上游 `merge_notes` 已标记商品冲突，写表层应优先保留本次 bundle 的商品主值，并把冲突说明保留到摘要字段，而不是把旧商品直接写进主表。**

### session_link -> 试衣Session索引（可选）
- `session_id`
- `link_action`
- `link_confidence`
- `link_reason`
- `needs_confirmation`
- `bundle_id`
- `message_ids`
- `bundle_status`
- `bundle_window_seconds`

## dedupe 建议
### 主表 dedupe_key
主表建议使用联合主键或联合去重键：
- `record_id`

若上游未显式生成 `record_id`，可退化为联合键：
- `session_id + customer_id + product_code`

### 素材表 dedupe_key
- `media_id`

### session 索引表 dedupe_key
- `session_id + bundle_id`

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
- **若 `write_decision.allow_write = false`，则连接器/写入器必须停止向主表写入；其中当 `write_decision.write_block_reason = PENDING_MEDIA_NOT_READY_TO_WRITE` 时，应直接跳过正式记录创建。**
- **若 `input_bundle.bundle_status = pending_media` 且 `fitting_records = []`，则不得向主表 `试衣商品记录` 写入任何正式记录；此时仅允许输出素材索引或 session 索引层面的待处理信息。**
- **若上游只提供单张图片且无明确文本，不得在 `sync_summary` 或任何落表字段中推断“成交/已购买/已完成”。**
- 输出应便于后续由连接器、脚本或多维表格写入器直接执行。

## 输出格式
返回：
```json
{"sync_result": {
  "tables": {
    "试衣商品记录": [],
    "试衣素材索引": [],
    "试衣Session索引": []
  },
  "write_mode": "upsert",
  "dedupe_keys": {
    "试衣商品记录": ["record_id"],
    "试衣素材索引": ["media_id"],
    "试衣Session索引": ["session_id", "bundle_id"]
  },
  "sync_status": "ready | partial | pending_confirm | no_binding",
  "sync_summary": "..."
}}
```

## 共享表建议
- 多导购共用一套飞书表时，主表必须至少保留 `guide_name`、`operator_id`、`operator_name`、`source_chat_id`、`source_sender_id`、`session_id` 这些区分字段。
