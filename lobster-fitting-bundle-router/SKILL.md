---
name: lobster-fitting-bundle-router
description: 将飞书会话中的图片、语音、文本在短时间窗口内归并为同一个试衣输入 bundle。适用于导购先发吊牌图、再发语音/文字补充的场景，用于避免一条消息一处理造成的上下文串单。
---

# 试衣输入归并路由

## 目标
把同一导购在短时间窗口内发送的多条消息，归并成一个 `bundle`，供 draft 管理和后续卡片预填使用。

## 输入
- `messages`
- `sender_id`
- `chat_id`
- `existing_open_bundle`（可选）
- `existing_open_draft`（可选）
- `context_policy`（可选，`inherit_if_safe` / `strict_isolation`）
- `reset_context`（可选，`true` 时表示当前消息应与旧 bundle 隔离）
- `force_new_bundle`（可选，`true` 时必须新建 bundle）

## 核心规则
- 默认归并窗口：`120s`
- 同一导购、同一会话、在时间窗口内的图 / 音 / 文优先归入同一个 `bundle`
- 单张吊牌图先进入 `pending_media`
- 若窗口内补充了语音或文本，则 `bundle_status` 更新为 `merged`
- 若超时无补充，则保留 `pending_media` 并交给 draft-manager 决定是否继续等待或提醒补充
- 不得仅因“最近一条记录存在”，就自动把新图片归到历史正式记录
- **若 `context_policy = strict_isolation`，应默认降低 `existing_open_bundle` / `existing_open_draft` 的续接优先级；除非当前消息带有明确补充语义，或时间窗口与业务主值都高匹配，否则优先 `new_bundle`**
- **若 `reset_context = true` 或 `force_new_bundle = true`，则必须输出 `routing_action = new_bundle`，不得 `append_bundle`**
- **若当前消息与 `existing_open_bundle` 跨自然日，即使仍处于 open 状态，也不得续接到旧 bundle；skill 层应直接按新 bundle 处理**

## 输出目标
返回 `result`：
- `bundle_id`
- `bundle_status` (`pending_media` / `single` / `merged`)
- `draft_id`
- `message_ids`
- `message_types`
- `window_seconds`
- `routing_action` (`append_bundle` / `new_bundle` / `keep_waiting`)
- `routing_reason`

## 输出格式
```json
{"result": {...}}
```
