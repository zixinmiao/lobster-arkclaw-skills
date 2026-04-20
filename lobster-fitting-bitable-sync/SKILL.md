---
name: lobster-fitting-bitable-sync
description: 将试衣合并结果映射成适合飞书多维表格写入的结构。适用于需要把“单客单商品单次试衣”记录 upsert 到飞书多维表格台账、补齐会员字段、保留 session 归属与素材映射、生成写入摘要时使用。
---

# 试衣飞书多维表格沉淀

把结构化试衣数据整理成适合飞书多维表格写入的结果。

## 依赖
当任务涉及飞书多维表格字段设计、记录写入、视图或公式语义时，优先配合 `lark-base` 类能力执行。

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
- 缺 binding 时，先输出待写入载荷，不伪造写入成功。
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
