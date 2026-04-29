---
name: lobster-followup-reminder-dispatch
description: 扫描线索回访表中的到期待触达记录，生成导购提醒内容，并回写提醒状态。适用于由统一定时任务周期性触发，避免为每条回访线索手动配置单独提醒。
---

# 回访提醒派发

## 目标
把“每条线索手动配置定时提醒”改成“统一定时任务 + 扫描派发”。

## 输入
- `current_time`
- `followup_leads`
- `dispatch_window_minutes`（可选，默认 15）
- `dispatch_mode`（可选，默认 `scheduled_due`）
- `channel_context`（可选）

### followup_leads
数组；每条记录尽量包含：
- `线索ID`
- `导购名称`
- `导购open_id`
- `会员尾号`
- `试穿单品`
- `回访原因`
- `建议触达时机类型`
- `计划触达时间`
- `建议回访内容`
- `推荐动作`
- `优先级`
- `状态`
- `是否已提醒`
- `提醒时间`

## 扫描规则
默认模式 `dispatch_mode = scheduled_due` 下，满足以下条件时进入派发候选：
- `状态 = pending`
- `计划触达时间 <= current_time`
- `是否已提醒 = false`

## 输出目标
返回 `result`：
- `due_reminders`
- `status_patches`
- `dispatch_summary`

### due_reminders
尽量包含：
- `线索ID`
- `导购名称`
- `target_chat_id`
- `reminder_title`
- `reminder_text`
- `priority`
- `send_now`
- `reason`

### status_patches
用于回写线索回访表，尽量包含：
- `线索ID`
- `状态`
- `是否已提醒`
- `提醒时间`
- `dispatch_note`

规则：
- 只要本次提醒已经成功发出，就立即回写 `是否已提醒 = true`
- 采用一次提醒制时，后续扫描应直接跳过 `是否已提醒 = true` 的记录

## 输出格式
```json
{"result": {...}}
```
