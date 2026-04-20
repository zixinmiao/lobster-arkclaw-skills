---
name: lobster-fitting-draft-manager
description: 管理导购试衣录入的 draft 草稿上下文。适用于将飞书对话中的图、音、文输入先沉淀到草稿，再通过卡片确认后生成正式试衣记录的场景。
---

# 试衣 draft 草稿管理

## 目标
将聊天输入从“直接驱动正式记录”改为“先进入 draft 草稿”，隔离上下文污染。

## draft 核心定义
一个 `draft` 代表一次待确认的试衣录入草稿，只有在卡片确认提交后，才可生成正式 `fitting_record`。

## draft 状态
- `pending_media`：只有图片，等待补充文本/语音
- `pending_feedback`：已有商品候选，等待试穿反馈
- `ready_for_card`：已具备卡片预填条件
- `confirmed`：卡片已确认
- `submitted`：已正式写入
- `cancelled`：取消本次草稿

## 输入
- `bundle_result`
- `ocr_result`（可选）
- `stt_result`（可选）
- `member_result`（可选）
- `existing_draft`（可选）

## 核心规则
- 每次试衣录入先创建或更新 `draft_id`
- 新吊牌图、新语音、新文本都先挂到 draft，不得直接生成正式记录
- draft 必须明确绑定：
  - `draft_id`
  - `bundle_id`
  - `product_candidates`
  - `feedback_candidates`
  - `media_records`
- 单张图无文本时，只允许进入 `pending_media`
- 若已有 OCR 候选商品 + 语音/文本反馈，则状态可转为 `ready_for_card`
- 只有卡片确认提交后，draft 才可转为 `confirmed -> submitted`

## 输出目标
返回 `result`：
- `draft_id`
- `draft_status`
- `draft_action` (`create` / `update` / `wait` / `ready_for_card` / `submit`)
- `product_candidates`
- `feedback_candidates`
- `member_candidate`
- `media_records`
- `write_gate`

### write_gate
- `allow_write` (`true` / `false`)
- `reason`

## 强约束
- `draft_status != confirmed` 时，`allow_write` 必须为 `false`
- 单图无文本时，`reason` 必须为 `PENDING_MEDIA_NOT_READY_TO_WRITE`
- 未经过卡片确认，不得生成正式 `fitting_record`

## 输出格式
```json
{"result": {...}}
```
