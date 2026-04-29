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
- `guide_profile` / `sender_profile`（可选）
- `existing_draft`（可选）
- `context_policy`（可选，`inherit_if_safe` / `strict_isolation`，默认建议按 `strict_isolation` 理解）
- `reset_context`（可选，`true` 表示本次明确清除上一条草稿上下文）
- `user_intent`（可选，如 `restart_current_fitting` / `ignore_previous_context`）

## 核心规则
- 每次试衣录入先创建或更新 `draft_id`
- 新吊牌图、新语音、新文本都先挂到 draft，不得直接生成正式记录
- draft 在创建或更新时，应优先继承 `guide_profile` / `sender_profile` 中的稳定字段，如 `guide_name`、`store_name`
- 对同一 sender，若这些字段此前已经确认，不应在后续每条录入里重新追问
- draft 必须明确绑定：
  - `draft_id`
  - `bundle_id`
  - `product_candidates`
  - `feedback_candidates`
  - `media_records`
- 单张图无文本时，只允许进入 `pending_media`
- 若已有 OCR 候选商品 + 语音/文本反馈，则状态可转为 `ready_for_card`
- 只有卡片确认提交后，draft 才可转为 `confirmed -> submitted`
- **若 `context_policy = strict_isolation`，应默认把历史 draft 视为弱参考，不得因为存在 `existing_draft` 就直接续写；只有出现明确延续信号时，才允许更新旧 draft，否则优先新建或返回待确认**
- **若 `reset_context = true`，或 `user_intent` 明确表达“重新开始 / 清掉上一条 / 不要接着刚才 / 忽略前文”，则本次必须将当前 `existing_draft` 视为失效上下文：输出 `draft_action = reset` 或将旧 draft 标记为 `cancelled`，并基于当前输入重新开始判断，不得继续挂靠旧 draft**
- **若当前输入与历史 draft 的商品主值、会员身份或日期冲突，应优先保留当前输入，并在 `merge_notes` / `write_gate.reason` 中说明“历史上下文已降权”**

## 输出目标
返回 `result`：
- `draft_id`
- `draft_status`
- `draft_action` (`create` / `update` / `wait` / `ready_for_card` / `submit` / `reset`)
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
- **若 `reset_context = true` 或识别到“重新开始 / 清掉上一条 / 不要关联前文”等明确重开意图，`draft_action` 必须输出 `reset`，且 `write_gate.reason` 固定输出 `CONTEXT_RESET_BY_USER`**
- **若当前输入日期与 `existing_draft` 所属日期不同，则不得因 `existing_draft` 存在而直接 `update`；应优先 `create` / `reset` / `wait`，实现 skill 内的跨天隔离**

## 输出格式
```json
{"result": {...}}
```
