---
name: lobster-followup-reminder-dispatch
description: 扫描线索回访表中的到期待触达记录，生成导购提醒内容，并回写提醒状态。适用于由统一定时任务周期性触发，避免为每条回访线索手动配置单独提醒。
---

# 回访提醒派发

## 目标
把“每条线索手动配置定时提醒”改成“统一定时任务 + 扫描派发”。

本 skill 不负责自己定时唤醒；它负责在被定时任务触发后：
1. 读取线索回访表中到期记录
2. 生成导购提醒内容
3. 输出待发送提醒与回写状态更新

## 关键边界
- skill 负责：判断哪些记录现在该提醒、提醒什么、发给谁
- 定时器负责：按固定频率触发 skill
- 因此推荐只手动配置 **1 个全局定时任务**，而不是为每条线索配置单独任务

## 输入
- `current_time`
- `followup_leads`
- `dispatch_window_minutes`（可选，默认 15）
- `dispatch_mode`（可选，默认 `scheduled_due`；可选 `weekly_event_review`）
- `channel_context`（可选，如飞书 chat/user mapping）

### followup_leads
数组；每条记录尽量包含：
- `followup_lead_id`
- `guide_name`
- `guide_chat_id`（可选）
- `member_mobile_last4`
- `product_name`
- `lead_reason`
- `planned_touch_at`
- `touch_time_type`
- `touch_trigger_condition`
- `followup_content`
- `recommended_action`
- `priority`
- `status`
- `is_reminded`
- `reminded_at`

## 扫描规则
### 一、什么记录应该进入本次提醒
默认模式 `dispatch_mode = scheduled_due` 下，满足以下条件时进入派发候选：
- `status = pending`
- `planned_touch_at <= current_time`
- `is_reminded = false`

### 二、触发式线索
对 `event_triggered` 类型，推荐使用 `dispatch_mode = weekly_event_review`：
- `touch_time_type = event_triggered`
- `status = pending` 或 `status = waiting_trigger`
- `is_reminded = false`
- 由每周固定任务统一拉出“待判断清单”提醒一次

### 三、去重与保护
- 同一 `followup_lead_id` 在同一调度窗口内不得重复提醒
- 同一导购短时间内若命中过多提醒，可按优先级排序后分批输出
- 若记录信息不足以安全提醒，应进入 `manual_review` 而不是强行发送

## 输出目标
返回 `result`：
- `due_reminders`
- `status_patches`
- `dispatch_summary`

### due_reminders
数组；每项代表一条待发送提醒。

尽量包含：
- `followup_lead_id`
- `guide_name`
- `target_chat_id`
- `reminder_title`
- `reminder_text`
- `priority`
- `send_now`
- `reason`

### 推荐提醒文案结构
提醒文案建议包含：
- 会员尾号
- 来源试穿商品
- 为什么现在该跟
- 建议跟什么内容
- 当前动作建议

例如：
- `会员尾号 1234 试穿了黑色西装外套，当时反馈喜欢但想再考虑一下。建议今天晚些时候跟进，重点发送同款上身效果和一套替代搭配建议。`

### 强约束：提醒文案不得放大或编造经营事实
- 若上游线索里没有明确活动、折扣、到货、库存、预留资格等真实信息，提醒文案不得出现此类内容。
- 提醒 skill 只负责转述已存在的结构化事实，不负责补写新的活动话术。
- 若上游 `followup_content` 已包含疑似编造信息，应优先降级为保守提醒，例如：`请根据门店当前真实活动和库存情况再联系会员。`

### status_patches
数组；用于回写线索回访表。

尽量包含：
- `followup_lead_id`
- `new_status` (`reminded` / `manual_review` / `waiting_trigger`)
- `is_reminded` (`true` / `false`)
- `reminded_at`
- `dispatch_note`

推荐规则：
- 只要本次提醒已经成功发出，就立即回写 `is_reminded = true`
- 采用“一次提醒制”时，后续扫描应直接跳过 `is_reminded = true` 的记录

### dispatch_summary
- `due_count`
- `sent_count`
- `manual_review_count`
- `waiting_trigger_count`

## 建议调度方式
推荐配置两个统一调度：
- 每 10 / 15 / 30 分钟执行一次 `scheduled_due`，扫描已到具体提醒时间且 `is_reminded = false` 的记录
- 每周固定执行一次 `weekly_event_review`，扫描 `event_triggered` 且 `is_reminded = false` 的记录，输出“待判断清单”
- 对 `due_reminders` 逐条发消息给导购
- 再按 `status_patches` 回写状态

## 输出格式
```json
{"result": {...}}
```
