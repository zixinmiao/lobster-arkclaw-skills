---
name: lobster-fitting-record-merge
description: 合并导购文字、语音转写、吊牌识别、会员识别和上下文，生成标准试衣主记录、商品明细和素材索引。适用于一次试衣采集链路中需要把多来源信息整理成可入库或可写飞书多维表格的数据时使用。
---

# 试衣记录合并

将多来源输入合并成标准结构。

## 输入来源
接受任意组合：
- 导购原始文字
- 语音转写结果
- 吊牌识别结果
- 会员识别结果
- 上下文中的门店、导购、客户、消息来源
- 图片/语音素材索引

## 输出目标
输出 `result`：
- `session_record`
- `item_records`
- `media_records`
- `merge_notes`
- `missing_fields`

### session_record
尽量包含：
- `session_id`
- `guide_name`
- `store_name`
- `fitting_date`
- `fitting_time`
- `is_member`
- `member_mobile_last4`
- `member_identity_status`
- `notes`
- `source_channel`
- `source_message_id`

### item_records
数组；每项尽量包含：
- `sku`
- `product_code`
- `product_name`
- `color`
- `size`
- `tag_price`
- `ocr_confidence`

### media_records
数组；每项尽量包含：
- `media_id`
- `media_type`
- `media_url`
- `media_source`

## 合并规则
- 同一字段有多个候选时，优先明确、完整、置信度更高的值。
- 不确定时保留冲突说明到 `merge_notes`。
- 不得编造 session_id 之外的业务事实；`session_id` 可按当前任务生成唯一标识。
- 若商品有多件，拆成多条 `item_records`。

## 输出格式
返回：
```json
{"result": {...}}
```
