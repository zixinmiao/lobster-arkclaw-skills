---
name: lobster-fitting-record-merge
description: 合并导购文字、语音转写、吊牌识别、会员识别、飞书拆分消息和上下文，生成以“单客单商品单次试衣”为粒度的标准记录。适用于一次试衣采集链路中需要先完成消息归并、session 判断、商品归属，再输出可入库或可写飞书多维表格的数据时使用。
---

# 试衣记录合并

将多来源输入先归并，再判断 session 归属，最后输出标准结构。

## 适用场景
- 飞书中图片和文本分两条消息发送，需要先归并再判断归属
- 需要区分“补充上一条”还是“新建一条试衣记录”
- 需要把输出粒度从“session 主记录 + 商品明细”调整为“单客单商品单次试衣一条主记录”

## 核心定义
### 1. 记录粒度
本 skill 的主记录粒度定义为：

> 同一客人在同一次试衣 session 中试穿的一件商品，记为一条 `fitting_record`

若同一次 session 中试穿 3 件商品，则输出 3 条 `fitting_records`。

### 2. session 定义
`session` 表示一次连续的试衣会话；它是上下文归属单位，不再直接等同于最终写入主表的一条记录。

### 3. bundle 定义
`bundle` 表示同一导购在短时间窗口内连续发送的多条消息组合，常见为：
- 先发吊牌图，再发文字
- 先发文字，再发图片
- 连续多张图片 + 一段补充描述

飞书场景下，**不要把单条图片消息直接视为一条完整试衣输入**，应优先先组成 `bundle` 后再做归属判断。

## 输入来源
接受任意组合：
- 导购原始文字
- 语音转写结果
- 吊牌识别结果
- 会员识别结果
- 上下文中的门店、导购、客户、消息来源
- 图片/语音素材索引
- 最近一次 open session 或候选 session 信息
- 飞书拆分消息列表（可选）

## 输入建议结构
### messages
可选；用于飞书拆分消息归并。

每条消息尽量包含：
- `message_id`
- `message_type` (`text` / `image` / `audio` / `mixed_ref`)
- `sender_id`
- `sent_at`
- `text`
- `media_id`
- `media_url`

### recent_sessions
可选；用于判断是否续写上一条。

每条 session 尽量包含：
- `session_id`
- `session_status` (`open` / `closed` / `pending_confirm`)
- `guide_name`
- `store_name`
- `member_mobile_last4`
- `last_active_at`
- `fitting_date`
- `product_codes`

## 输出目标
输出 `result`：
- `input_bundle`
- `session_link`
- `fitting_records`
- `media_records`
- `write_decision`
- `merge_notes`
- `missing_fields`

### input_bundle
描述归并后的输入包。

尽量包含：
- `bundle_id`
- `bundle_status` (`merged` / `single` / `pending_media`)
- `bundle_window_seconds`
- `message_ids`
- `primary_text`
- `media_count`

### session_link
描述本次 bundle 对哪个 session 生效。

尽量包含：
- `link_action` (`append` / `new_session` / `pending_confirm`)
- `target_session_id`
- `link_confidence`
- `link_reason`
- `needs_confirmation`

### fitting_records
数组；每项代表一条最终主记录，粒度为“单客单商品单次试衣”。

每项尽量包含：
- `record_id`
- `session_id`
- `session_status`
- `fitting_date`
- `fitting_time`
- `guide_name`
- `store_name`
- `customer_id`
- `is_member`
- `member_mobile_last4`
- `member_identity_status`
- `sku`
- `product_code`
- `product_name`
- `color`
- `size`
- `tag_price`
- `ocr_confidence`
- `try_on_result`
- `body_effect_desc`
- `fit_feedback`
- `liked_points`
- `disliked_points`
- `not_buy_reason`
- `followup_intent`
- `is_high_intent`
- `source_channel`
- `source_bundle_id`
- `source_message_ids`
- `raw_notes`

### media_records
数组；每项代表一条素材记录。

每项尽量包含：
- `media_id`
- `record_id`
- `session_id`
- `product_code`
- `media_type`
- `media_url`
- `media_source`
- `message_id`
- `captured_at`

### write_decision
描述当前结果是否允许写入正式业务主表。

尽量包含：
- `allow_write` (`true` / `false`)
- `write_block_reason`
- `write_target` (`正式主表` / `仅素材索引` / `仅缓存等待`)

## 关键规则

### A. 先归并，再判断
- 先把短时间窗口内、同一导购发出的图片/文本消息归并为 `input_bundle`
- 默认归并窗口建议为 **120 秒**
- 若只有单张图片且暂无文本，先标记为 `pending_media`，不要立刻归入上一条正式记录
- **硬约束：当 `bundle_status = pending_media` 且缺少可用文本描述时，`fitting_records` 必须返回空数组 `[]`；此时只允许保留 `media_records`、`missing_fields`、`merge_notes` 和待确认状态，不得生成正式试衣记录**
- **同一场景下，`write_decision.allow_write` 必须为 `false`，且 `write_decision.write_block_reason` 必须固定输出 `PENDING_MEDIA_NOT_READY_TO_WRITE`，供上层连接器直接拦截写入**

### B. session 判断规则
#### 允许 `append` 到已有 session 的条件
需高置信满足多个条件，例如：
- 同一天
- 最近存在 `open` 状态 session
- 时间间隔较短
- 文本中有明显延续语义，如“补充上一条”“继续刚才那件”
- 商品编码、会员信息或上下文与当前 open session 高匹配

#### 默认 `new_session` 的条件
满足任一情况时，优先新建 session：
- **跨自然日**
- 当前不存在 open session
- 商品信息与上一条明显不同
- 上一条 session 已 `closed`
- 当前 bundle 更像一次新的录入开始，而不是补充说明

#### `pending_confirm` 条件
以下情况不要硬归属：
- 只有图片，没有足够文本
- 同一天但距离上一条较久，且缺少明确延续信号
- 商品识别结果不稳定或冲突
- 无法判断是补充上一条，还是新的一条

### C. 飞书拆分消息规则
- 飞书中图片与文本分开发送时，应把它们视为一次可能的组合输入，而不是独立记录
- **不得仅因“最近一条记录存在”，就把新图片自动归入最近一条记录**
- 若图片先到、文本后到，应优先等待短窗口内补充文本，再整体判断归属
- **单张吊牌图只能支持“识别到商品候选信息”，不能单独推断“成交/已购买/已完成试衣记录”**
- **成交判断必须来自明确文本、结构化确认字段，或后续补充后的完整 bundle；不得仅依据图片直接生成成交结论**

### D. 数据粒度规则
- 最终主记录使用 `fitting_records` 输出
- 一件商品 + 一个客人 + 一个 session = 一条主记录
- 若同一 session 内有多件商品，拆成多条 `fitting_records`
- `session` 是归属单位，不等于主表记录本身

### E. 编造限制
- 不得编造 `record_id` / `session_id` 之外的业务事实
- `record_id` 与 `session_id` 可按当前任务生成唯一标识
- 若商品、客户、反馈信息不完整，可输出部分字段，但必须在 `missing_fields` / `merge_notes` 中说明
- **在只有图片、没有明确文本反馈时，不得编造成交、试穿结果、是否购买、跟进意向等业务结论**

## 输出格式
返回：
```json
{"result": {
  "input_bundle": {...},
  "session_link": {...},
  "fitting_records": [],
  "media_records": [],
  "write_decision": {
    "allow_write": false,
    "write_block_reason": "PENDING_MEDIA_NOT_READY_TO_WRITE",
    "write_target": "仅缓存等待"
  },
  "merge_notes": ["PENDING_MEDIA_NOT_READY_TO_WRITE"],
  "missing_fields": ["text_feedback"]
}}
```
