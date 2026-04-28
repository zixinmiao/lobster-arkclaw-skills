# 默认建表结构

在执行建表或补字段前，先按真实表结构核对；以下是默认最小 schema。

## 1. 试衣商品记录
用于沉淀“单客单商品单次试衣”主记录。

建议字段：
- `record_id`（文本）
- `session_id`（文本）
- `session_status`（文本）
- `fitting_date`（日期/日期时间）
- `fitting_time`（日期/日期时间）
- `guide_name`（文本）
- `store_name`（文本）
- `operator_id`（文本）
- `operator_name`（文本）
- `source_chat_id`（文本）
- `source_sender_id`（文本）
- `brand_name`（文本）
- `customer_id`（文本）
- `is_member`（复选框/布尔）
- `member_mobile_last4`（文本）
- `member_identity_status`（文本）
- `sku`（文本）
- `product_code`（文本）
- `product_name`（文本）
- `color`（文本）
- `size`（文本）
- `tag_price`（数字）
- `ocr_confidence`（数字）
- `try_on_result`（文本）
- `body_effect_desc`（长文本）
- `fit_feedback`（长文本）
- `liked_points`（长文本）
- `disliked_points`（长文本）
- `not_buy_reason`（长文本）
- `followup_intent`（文本）
- `is_high_intent`（复选框/布尔）
- `source_channel`（文本）
- `source_bundle_id`（文本）
- `source_message_ids`（长文本）
- `raw_notes`（长文本）

建议去重键：
- `record_id`

## 2. 试衣素材索引
用于沉淀图片、语音等素材与主记录映射。

建议字段：
- `media_id`（文本）
- `record_id`（文本）
- `session_id`（文本）
- `product_code`（文本）
- `media_type`（文本）
- `media_url`（超链接/文本）
- `media_source`（文本）
- `message_id`（文本）
- `captured_at`（日期/日期时间）

建议去重键：
- `media_id`

## 3. 试衣Session索引
用于追踪 session 生命周期和 bundle 归属。

建议字段：
- `session_id`（文本）
- `link_action`（文本）
- `link_confidence`（数字）
- `link_reason`（长文本）
- `needs_confirmation`（复选框/布尔）
- `bundle_id`（文本）
- `message_ids`（长文本）
- `bundle_status`（文本）
- `bundle_window_seconds`（数字）

建议去重键：
- `session_id`
- `bundle_id`

## 建表原则
- 先建最小可用字段，不在 bootstrap 阶段引入复杂公式字段
- 字段类型以当前飞书 Base 可稳定写入为优先
- 主表必须保留导购/录入来源识别字段，确保多导购共用同一张表时仍可区分记录归属
- 真实写入前，以上游 `lobster-fitting-bitable-sync` 的字段映射为准
- 若已有历史表结构，不强制重建，优先补齐缺失字段
