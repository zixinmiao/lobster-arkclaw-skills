---
name: lobster-fitting-card-fill
description: 根据 draft、OCR、STT 和 merge 结果生成飞书试衣录入卡片的预填内容，并接收卡片确认后的结构化结果。适用于“会话采集图音 + 卡片确认提交”的试衣录入方案。
---

# 试衣录入卡片预填与确认

## 目标
把 draft 中的候选字段整理成飞书卡片可展示、可编辑、可提交的结构。

## 输入
- `draft_result`
- `merge_result`
- `ocr_result`
- `stt_result`

## 卡片字段建议
### 状态区
- `draft_id`
- `status_text`
- `source_summary`

### 商品信息区
- `product_name`
- `product_code`
- `sku`
- `color`
- `size`
- `tag_price`
- `tag_image_url`

### 试穿反馈区
- `body_effect_desc`
- `liked_points`
- `disliked_points`
- `not_buy_reason`
- `is_purchased`
- `is_high_intent`
- `need_followup`

### 基础区
- `member_mobile_last4`
- `guide_name`
- `store_name`
- `fitting_time`

### 动作区
- `save_draft`
- `submit_record`
- `continue_add_item`
- `cancel_draft`

## 预填规则
- 商品区优先使用本次 OCR 结果
- 反馈区优先使用本次 STT / 文本抽取结果
- 若字段置信度低，显示“待确认”提示
- 卡片展示的是 draft 候选值，不是最终正式记录

## 提交规则
- 点击 `submit_record` 后，输出结构化确认结果
- 该结果可作为正式 `fitting_record` 的唯一提交输入
- 提交前若关键字段缺失，应返回阻断原因，不得直接写入

## 输出目标
返回 `result`：
- `card_payload`
- `card_state`
- `submit_payload_schema`
- `blocking_rules`

## 输出格式
```json
{"result": {...}}
```
