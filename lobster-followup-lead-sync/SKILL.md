---
name: lobster-followup-lead-sync
description: 根据试衣反馈判断是否生成回访线索，并同步更新试衣表与线索回访表。适用于试衣记录已形成，需要把“是否回访、何时触达、为什么触达、回访做什么”结构化落表的场景。
---

# 试衣回访线索生成与落表

## 目标
基于单条试衣记录，输出两层结果：
1. 是否需要生成回访线索，并回写到现有试衣表
2. 若需要，则生成一条“线索回访表”记录，供后续定时提醒与导购跟进使用

## 适用前提
当前试衣表已有基础字段：
- `日期`
- `导购`
- `会员尾号`
- `试穿单品`
- `客户反馈`
- `结果`
- `客户体型描述`

## 输入
- `fitting_record`
- `member_context`（可选）
- `store_context`（可选）
- `current_time`（可选）
- `campaign_context`（可选，如活动、折扣、到货信息）

### fitting_record
尽量包含：
- `fitting_record_id`
- `fitting_date`
- `guide_name`
- `member_mobile_last4`
- `product_name`
- `product_code`
- `customer_feedback`
- `try_on_result`
- `body_shape_desc`
- `liked_points`（可选）
- `disliked_points`（可选）
- `not_buy_reason`（可选）

## 核心判断逻辑
### 一、什么情况下生成回访线索
满足“有兴趣 + 未成交 + 原因明确 + 后续可被导购动作解决”时，优先生成线索。

典型信号：
- 明显高意向但未成交：如“喜欢、挺合适、再考虑一下、回去想想”
- 阻碍可解：如缺码、缺色、预算卡点、等活动、想看替代款
- 偏好明确：如已明确风格、版型、颜色偏好，可支撑二次推荐
- 用户授权后续联系：如“到货通知我、活动提醒我、有类似款发我”

### 二、什么情况下不生成高优先级回访线索
以下情况默认不生成，或仅保留为低优先级：
- 明确表达不喜欢
- 明确不适合自己风格
- 明确近期无购买计划
- 只是礼貌性试穿，无兴趣延续
- 阻碍不可逆，短期内无可行动作

### 三、如何确定建议触达时机
触达时机由“未成交原因”决定，而不是识别出线索后统一立即触达。

推荐规则：
- `same_day`：离店后 2~6 小时；适合高意向未成交
- `next_day`：次日中午或晚上；适合“回去考虑一下”
- `event_triggered`：到货、补码、活动、新款触发；适合缺码/等活动/等替代款
- `weekend_or_holiday`：周末前、节前；适合持续兴趣但非立即决策型
- `not_recommended`：不建议立即跟进，仅沉淀偏好

### 四、具体提醒时间的硬规则
对以下类型，必须生成具体 `planned_touch_at` / `recommended_touch_time`，不得留空：
- `same_day`
- `next_day`
- `weekend_or_holiday`

默认生成规则：
- `same_day`：默认取 `current_time + 4 小时`；若结果晚于当天 `20:00`，则取当天 `20:00`
- `next_day`：默认取次日 `12:00`
- `weekend_or_holiday`：默认取最近一个周六 `14:00`

例外：
- `event_triggered`：允许无具体 `planned_touch_at`，由后续每周固定扫描“待判断清单”
- `manual_confirm` / `not_recommended`：允许为空

## 强约束：不得编造活动、折扣与经营事实
- 若输入里没有明确的 `campaign_context` / `arrival_context` / `inventory_context`，不得在回访建议中编造任何活动、折扣、会员日、限时价、预留库存、到货时间等信息。
- “建议触达时机”可以输出为抽象类型，如 `next_day`、`weekend_or_holiday`、`event_triggered`；其中 `same_day` / `next_day` / `weekend_or_holiday` **必须**进一步落成一个**默认具体提醒时间**，供 reminder skill 扫描；`event_triggered` 可保持无具体时间，由后续统一扫描提醒。
- `recommended_followup_content` 和 `followup_content` 应优先使用保守表达，例如：
  - `您上次试穿的这条裤子版型很适合您，如果您愿意，我可以再帮您看看有没有更合适的搭配或近期合适的入手时机。`
  - `如果后续这款有合适的尺码/颜色/活动信息，我第一时间联系您。`
- 若缺少真实经营信息，允许输出“建议后续补充活动/到货信息后再触达”，但不得把假信息写成既成事实。

## 输出目标
返回 `result`：
- `fitting_table_patch`
- `followup_lead`
- `decision_reason`

### fitting_table_patch
用于回写现有试衣表的增量字段。

尽量包含：
- `fitting_record_id`
- `should_create_followup_lead`
- `followup_reason`
- `recommended_touch_time_type`
- `recommended_touch_time`
- `touch_trigger_condition`
- `recommended_followup_content`
- `followup_priority`
- `should_update_member_profile`
- `member_profile_delta_summary`

说明：
- 对 `same_day` / `next_day` / `weekend_or_holiday`，**必须**输出默认具体提醒时间，便于后续 reminder skill 直接扫描；未生成具体时间视为结果不完整。
- 对 `event_triggered`，`recommended_touch_time` 允许为空，由后续每周固定扫描一次“待判断清单”。
- `recommended_followup_content` 必须可追溯到输入事实，不得包含未给定的活动名、折扣价、库存承诺。

### followup_lead
若需生成回访线索，则输出一条结构化记录。

尽量包含：
- `followup_lead_id`
- `source_fitting_record_id`
- `guide_name`
- `member_mobile_last4`
- `product_name`
- `product_code`
- `lead_reason`
- `touch_time_type` (`same_day` / `next_day` / `event_triggered` / `weekend_or_holiday` / `manual_confirm`)
- `planned_touch_at`
- `touch_trigger_condition`
- `followup_content`
- `recommended_action`
- `priority`
- `status` (`pending` / `waiting_trigger` / `cancelled` / `reminded`)
- `is_reminded` (`true` / `false`)
- `reminded_at`
- `confidence`

说明：
- 若 `touch_time_type = event_triggered`，必须同时说明真实触发条件；没有触发条件时，降级为 `manual_confirm` 或更保守时机。
- 若 `touch_time_type` 为 `same_day` / `next_day` / `weekend_or_holiday`，则 `planned_touch_at` **必须有值**；为空视为不合格输出。
- 若采用“一次提醒制”，新生成的线索默认应写入 `is_reminded = false`、`reminded_at = null`。
- `followup_content` 只能基于已知事实生成，不得编造促销、活动、价格变化、库存预留等经营承诺。

### decision_reason
- `should_create`
- `why`
- `why_not`（可选）
- `missing_info`（可选）

## 线索回访表建议字段
建议最终落地为独立表：`线索回访表`

字段建议：
- `线索ID`
- `来源试衣记录ID`
- `日期`
- `导购`
- `会员尾号`
- `试穿单品`
- `是否需要回访`
- `回访原因`
- `建议触达时机类型`
- `计划触达时间`
- `触发条件`
- `建议回访内容`
- `推荐动作`
- `优先级`
- `状态`
- `是否已提醒`
- `提醒时间`
- `实际触达时间`
- `触达结果`

## 试衣表建议新增字段
在现有试衣表上追加：
- `是否生成回访线索`
- `回访原因`
- `建议触达时机`
- `触达触发条件`
- `建议回访内容`
- `线索优先级`
- `是否建议更新会员画像`
- `会员画像补充摘要`

## 输出格式
```json
{"result": {...}}
```
