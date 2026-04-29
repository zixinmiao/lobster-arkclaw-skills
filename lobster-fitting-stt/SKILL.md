---
name: lobster-fitting-stt
description: 将导购发送的试衣语音整理成结构化文字记录。适用于试衣记录采集场景中需要把语音描述转成可沉淀字段时使用，包括提取导购、门店、客户、商品、颜色、尺码、价格、试穿反馈、跟进动作等信息。
---

# 试衣语音转写

将输入中的语音内容转成结构化文本。

## 前置要求
本 skill 默认处理的是“已经拿到 transcript 的语音内容”，或“可直接读取的音频附件”。

推荐链路：
- 若输入是音频附件：先下载音频 -> 调用语音转写（如 Whisper）-> 再交给本 skill 整理结构化字段
- 不要在仅因“当前是录音消息”就直接回复“暂时无法直接转写语音”

如果转写失败，应输出明确失败原因，而不是模糊兜底：
- `audio_download_failed`
- `audio_decode_failed`
- `transcribe_failed`


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
