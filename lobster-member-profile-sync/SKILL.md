---
name: lobster-member-profile-sync
description: 基于试衣反馈补充会员画像，并同步写入独立的会员画像表。适用于已识别会员身份，需要把一次次试衣反馈沉淀为可复用会员偏好与后续经营信息的场景。
---

# 会员画像补充与同步

## 目标
从试衣反馈里抽取“可长期复用”的会员信息，沉淀为独立画像表，而不是全部堆回单次试衣记录。

## 字段 contract
写表前必须先读取 `../references/bitable-field-contract.md`，确认当前目标字段语义、最小可写字段集和可疑字段。

## 链接初始化机制
执行前优先使用人工提供链接做首次登记，不依赖预置共享配置：
- 先确认是否已有已保存的 `bitable_binding`
- 若无，则优先接收对应表链接并直接解析 `base_token / table_id / view_id`
- 再读取真实字段结构，确认目标表可用
- 只有在没有 binding、也没有链接时，才 fallback 搜索名称相近的候选表：`会员画像`、`导购小龙虾会员画像`
- 再读取字段结构，校验是否包含这些关键字段：
  - `会员尾号`
  - `风格偏好`
  - `版型偏好`
  - `价格敏感度`
  - `常见未购原因`
  - `最近更新时间`
- 若链接可用则直接登记 binding；若走 fallback 搜索，仍需人工确认，不自动直接写入

## 自动触发规则
以下情况默认应触发本 skill，而不是等人工额外指定：
- `is_member = true`
- 已识别出 `member_mobile_last4`
- 已有明确会员身份，且本次试衣产生了任一可沉淀信息：
  - `product_name`
  - `fit_feedback`
  - `body_effect_desc`
  - `liked_points`
  - `disliked_points`
  - `not_buy_reason`
  - `followup_intent`

如果识别到会员却没有触发本 skill，应视为流程漏调。

## 最低同步要求
即使当前证据不足以写满画像，也应尽量补最小画像更新：
- `member_mobile_last4`
- `last_profile_update_at`
- `profile_source_record_id`
- `profile_confidence`

也就是说：
- 强证据 → 补全偏好字段
- 弱证据 → 至少留下“这次会员被识别且发生过一次有效试衣”的画像痕迹

## 输入
- `fitting_record`
- `existing_member_profile`（可选）
- `current_time`（可选）

### fitting_record
尽量包含：
- `record_id`
- `member_mobile_last4`
- `is_member`
- `product_name`
- `fit_feedback`
- `try_on_result`
- `body_effect_desc`
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
建议输出这些字段语义，由发现后的真实字段承接：
- `profile_id`
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

### fitting_table_patch
用于回写试衣表，保留“本次补充了什么”。若主表还没加承接字段，可只在结果里输出，不强制写回。

### profile_sync_decision
- `should_sync`
- `why`
- `conflict_notes`（可选）
- `missing_info`（可选）

## 写表约束
- 会员画像表当前最稳的去重键是 `会员尾号`；没有明确会员尾号时，不要硬写长期画像。
- `最近更新时间` 若为文本字段，建议统一写 `YYYY-MM-DD HH:mm:ss`。
- 若只有单次弱信号，优先把内容收敛到 `体型备注 / 常见未购原因 / 画像置信度`，不要一次写满所有偏好字段。
- 若表还处于候选确认阶段，先输出候选绑定信息，不直接写业务记录。
- 识别到会员后，不允许静默跳过；至少要输出一份 `profile_table_patch` 或 `profile_sync_pending_reason`。

## 与试衣表的关系
- 试衣表保留单次事实
- 会员画像表沉淀长期偏好
- 每次试衣反馈可只回写一个 `member_profile_delta_summary` 到试衣表，避免主表过重

## 输出格式
```json
{"result": {...}}
```
