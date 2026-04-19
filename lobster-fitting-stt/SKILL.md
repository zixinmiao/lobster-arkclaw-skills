---
name: lobster-fitting-stt
description: 将导购发送的试衣语音整理成结构化文字记录。适用于试衣记录采集场景中需要把语音描述转成可沉淀字段时使用，包括提取导购、门店、客户、商品、颜色、尺码、价格、试穿反馈、跟进动作等信息。
---

# 试衣语音转写

将输入中的语音内容转成结构化文本。

## 输出目标
输出一个 JSON 对象 `result`，优先包含：
- `transcript`: 语音原文整理后的文本
- `guide_name`
- `store_name`
- `customer_name`
- `fitting_date`
- `fitting_time`
- `products`: 数组；每项尽量包含 `product_name` `product_code` `color` `size` `tag_price`
- `customer_feedback`
- `next_action`
- `notes`
- `missing_fields`: 数组；列出未识别字段

## 执行规则
- 保留业务事实，不补造未提及信息。
- 口语、重复、语气词可以整理，但不要改变原意。
- 有多个商品时，按商品拆分。
- 识别不确定信息时，用 `unknown` 或放入 `notes`，并写入 `missing_fields`。
- 如果语音中包含会员线索，不要自行定性，交由会员识别 skill 处理。

## 输出格式
返回：
```json
{"result": {...}}
```
