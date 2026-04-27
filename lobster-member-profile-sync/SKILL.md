---
name: lobster-member-profile-sync
description: 基于试衣反馈补充会员画像，并同步写入独立的会员画像表。适用于已识别会员身份，需要把一次次试衣反馈沉淀为可复用会员偏好与后续经营信息的场景。
---

# 会员画像补充与同步

## 目标
从试衣反馈里抽取“可长期复用”的会员信息，沉淀为独立画像表，而不是全部堆回单次试衣记录。

## 输入
- `fitting_record`
- `existing_member_profile`（可选）
- `current_time`（可选）

### fitting_record
尽量包含：
- `fitting_record_id`
- `member_mobile_last4`
- `is_member`
- `product_name`
- `customer_feedback`
- `try_on_result`
- `body_shape_desc`
- `liked_points`（可选）
- `disliked_points`（可选）
- `not_buy_reason`（可选）
- `followup_intent`（可选）

## 画像补充原则
### 一、补的是“动态经营画像”，不是基础静态资料
重点不是姓名、电话、生日这类基础档案，而是更贴近成交和回访的动态偏好。

### 二、优先提取可复用信息
优先沉淀以下几类：
- 风格偏好
- 版型偏好
- 尺码/体型适配规律
- 颜色偏好
- 面料偏好
- 价格敏感度
- 常见未购原因
- 回访触发条件
- 搭配需求 / 购物场景

### 三、只在有证据时更新
- 若只是弱猜测，不要强写长期画像
- 若当前输入只支持“本次观察”，可先进入 `profile_delta_summary`
- 若与历史画像冲突，应保留冲突说明，而不是直接覆盖

## 输出目标
返回 `result`：
- `profile_table_patch`
- `fitting_table_patch`
- `profile_sync_decision`

### profile_table_patch
用于写入独立会员画像表。

尽量包含：
- `member_profile_id`
- `member_mobile_last4`
- `style_preference`
- `fit_preference`
- `size_preference`
- `body_shape_notes`
- `color_preference`
- `material_preference`
- `price_sensitivity`
- `common_not_buy_reasons`
- `followup_trigger_preferences`
- `styling_needs`
- `profile_confidence`
- `last_profile_update_at`
- `profile_source_record_id`

### fitting_table_patch
用于回写试衣表，保留“本次补充了什么”。

尽量包含：
- `fitting_record_id`
- `should_update_member_profile`
- `member_profile_delta_summary`
- `profile_sync_status`

### profile_sync_decision
- `should_sync`
- `why`
- `conflict_notes`（可选）
- `missing_info`（可选）

## 会员画像表建议字段
建议新增独立表：`会员画像表`

字段建议：
- `会员画像ID`
- `会员尾号`
- `风格偏好`
- `版型偏好`
- `尺码偏好`
- `体型备注`
- `颜色偏好`
- `面料偏好`
- `价格敏感度`
- `常见未购原因`
- `回访触发偏好`
- `搭配需求`
- `最近更新时间`
- `最近来源试衣记录`
- `画像置信度`

## 与试衣表的关系
- 试衣表保留单次事实
- 会员画像表沉淀长期偏好
- 每次试衣反馈可只回写一个 `member_profile_delta_summary` 到试衣表，避免主表过重

## 输出格式
```json
{"result": {...}}
```
