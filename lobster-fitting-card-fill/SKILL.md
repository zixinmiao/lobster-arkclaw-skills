---
name: lobster-fitting-card-fill
description: 根据 draft、OCR、STT 和 merge 结果生成飞书试衣录入卡片的预填内容，并接收卡片确认后的结构化结果。适用于需要围绕当前保留字段做卡片确认提交的场景。
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
### 商品信息区
- `product_name`
- `product_code`
- `color`
- `size`
- `tag_price`

### 试穿反馈区
- `try_on_result`
- `body_effect_desc`
- `fit_feedback`
- `liked_points`
- `disliked_points`
- `not_buy_reason`
- `followup_intent`

### 基础区
- `session_id`
- `guide_name`
- `store_name`
- `fitting_date`
- `fitting_time`
- `is_member`
- `member_mobile_last4`

## 提交规则
- 点击 `submit_record` 后，输出结构化确认结果
- 提交前若 required 字段缺失，应返回阻断原因，不得直接写入
- 卡片提交结果应对齐当前主表保留字段，不再要求已被删减的字段

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
