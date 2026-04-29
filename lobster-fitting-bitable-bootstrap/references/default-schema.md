# 默认建表结构

在执行建表或补字段前，先按真实表结构核对；以下是默认最小 schema。

## 1. 试衣商品记录
用于沉淀单客单商品单次试衣主记录。

建议字段：
- `session_id`（文本）
- `guide_name`（文本）
- `session_status`（文本）
- `fitting_date`（日期/日期时间）
- `fitting_time`（日期/日期时间）
- `store_name`（文本）
- `operator_id`（文本）
- `is_member`（复选框/布尔）
- `member_mobile_last4`（文本）
- `product_code`（文本）
- `product_name`（文本）
- `color`（文本）
- `tag_price`（数字）
- `ocr_confidence`（数字）
- `try_on_result`（文本）
- `body_effect_desc`（长文本）
- `fit_feedback`（长文本）
- `liked_points`（长文本）
- `disliked_points`（长文本）
- `followup_intent`（文本）
- `source_channel`（文本）
- `source_bundle_id`（文本，飞书场景下应填发送人 open_id）
- `source_message_ids`（长文本/文本）
- `raw_notes`（长文本）
- `size`（文本）
- `not_buy_reason`（长文本）

建议去重键：
- `session_id`

必填校验字段：
- `session_id`
- `guide_name`
- `fitting_date`
- `fitting_time`
- `operator_id`
- `is_member`
- `product_code`
- `product_name`
- `try_on_result`
- `source_bundle_id`
- `raw_notes`

## 2. 试衣素材索引
用于沉淀图片、语音等素材与主记录映射。

建议字段：
- `media_id`（文本）
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

## 4. 线索回访表
用于沉淀基于试衣反馈生成的后续回访线索。

建议字段：
- `线索ID`（文本）
- `来源试衣记录ID`（文本）
- `日期`（日期/日期时间）
- `导购名称`（文本）
- `导购open_id`（文本）
- `会员尾号`（文本）
- `试穿单品`（长文本/文本）
- `回访原因`（长文本）
- `建议触达时机类型`（文本）
- `计划触达时间`（日期/日期时间）
- `建议回访内容`（长文本）
- `推荐动作`（长文本/文本）
- `优先级`（文本）
- `状态`（文本）
- `是否已提醒`（复选框/布尔）
- `提醒时间`（日期/日期时间）

建议去重键：
- `线索ID`

必填校验字段：
- `线索ID`
- `来源试衣记录ID`
- `日期`
- `导购名称`
- `导购open_id`
- `会员尾号`
- `试穿单品`
- `回访原因`
- `建议触达时机类型`
- `建议回访内容`
- `推荐动作`
- `优先级`
- `状态`
- `是否已提醒`

## 5. 会员画像表
用于沉淀会员长期可复用的偏好与经营信息。

建议字段：
- `会员尾号`（文本）
- `风格偏好`（长文本）
- `版型偏好`（长文本）
- `尺码偏好`（文本）
- `颜色偏好`（长文本）
- `面料偏好`（长文本）
- `价格敏感度`（文本/长文本）
- `常见未购原因`（长文本）
- `搭配需求`（长文本）
- `体型备注`（长文本）
- `最近更新时间`（日期/日期时间）
- `最近来源试衣记录`（文本）
- `画像置信度`（文本）

建议去重键：
- `会员尾号`

必填校验字段：
- `会员尾号`
- `最近更新时间`
- `最近来源试衣记录`
- `画像置信度`

## 建表原则
- 先建最小可用字段，不在 bootstrap 阶段引入复杂公式字段
- 字段类型以当前飞书 Base 可稳定写入为优先
- 真实写入前，以上游 contract 和 sync 规则为准
- 若已有历史表结构，不强制重建，优先补齐缺失字段
- required 字段缺失时，应阻断正式写入，而不是继续部分落表
