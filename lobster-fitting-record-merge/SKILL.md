---
name: lobster-fitting-record-merge
description: 合并导购文字、语音转写、吊牌识别、会员识别、上下文和消息元数据，生成适合稳定写入试衣主表的结构化记录。适用于一次试衣采集链路中需要先完成消息归并、session 判断和必填字段校验的场景。
---

# 试衣记录合并

将多来源输入先归并，再判断 session 归属，最后输出标准结构。

## 输入来源
接受任意组合：
- 导购原始文字
- 语音转写结果
- 吊牌识别结果
- 会员识别结果
- 上下文中的门店、导购、客户、消息来源
- `sender_profile` / `guide_profile`（可选）
- 图片/语音素材索引
- 最近一次 open session 或候选 session 信息
- 飞书拆分消息列表（可选）

## 输出目标
输出 `result`：
- `input_bundle`
- `session_link`
- `fitting_records`
- `media_records`
- `write_decision`
- `merge_notes`
- `missing_fields`

### fitting_records
每项尽量包含当前主表保留字段：
- `session_id`
- `guide_name`
- `session_status`
- `fitting_date`
- `fitting_time`
- `store_name`
- `operator_id`
- `is_member`
- `member_mobile_last4`
- `product_code`
- `product_name`
- `color`
- `tag_price`
- `ocr_confidence`
- `try_on_result`
- `body_effect_desc`
- `fit_feedback`
- `liked_points`
- `disliked_points`
- `followup_intent`
- `source_channel`
- `source_bundle_id`
- `source_message_ids`
- `raw_notes`
- `size`
- `not_buy_reason`

### write_decision
尽量包含：
- `allow_write`
- `write_block_reason`
- `write_target`

## 关键规则
### A. 先归并，再判断
- 先把短时间窗口内、同一导购发出的图片/文本消息归并为 `input_bundle`
- 默认归并窗口建议为 120 秒
- 若只有单张图片且暂无文本，先标记为 `pending_media`
- 当 `bundle_status = pending_media` 且缺少可用文本描述时，`fitting_records` 必须返回空数组，且 `write_decision.allow_write = false`

### B. 必填字段前置校验
要进入正式写表候选，以下字段必须可被稳定提取：
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

若任一字段缺失：
- 必须写入 `missing_fields`
- `write_decision.allow_write` 必须为 `false`
- `write_block_reason` 应明确为 `MISSING_REQUIRED_FIELDS`

### C. 导购资料继承规则
- 若当前消息未再次显式提供 `guide_name` / `store_name`，但 `sender_profile` 中已有稳定值，应直接继承
- `source_bundle_id` 应优先直接取自入站消息 metadata（如飞书 sender open_id），而不是由模型从正文中推断
- 不得因为单条消息缺少姓名/门店，就忽略同一 sender 在历史中已经明确给过的信息
- 只有在 `sender_profile` 不存在、或当前输入与历史资料冲突时，才允许将其标记到 `missing_fields`

### D. 商品主值约束
- `product_code` / `product_name` 应优先采用本次 bundle 的 OCR 结果 + 本次文本补充
- 若 OCR 结果与历史上下文商品冲突，应在 `merge_notes` 中记录冲突，并优先保留本次输入商品

### E. 编造限制
- 不得编造必填字段
- 在只有图片、没有明确文本反馈时，不得编造成交、试穿结果、是否购买、跟进意向等业务结论
- 若无法稳定拿到必填字段，应阻断写入，而不是部分落表

## 输出格式
返回：
```json
{"result": {
  "input_bundle": {},
  "session_link": {},
  "fitting_records": [],
  "media_records": [],
  "write_decision": {
    "allow_write": false,
    "write_block_reason": "MISSING_REQUIRED_FIELDS",
    "write_target": "仅缓存等待"
  },
  "merge_notes": [],
  "missing_fields": []
}}
```
