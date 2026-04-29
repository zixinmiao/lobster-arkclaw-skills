# bitable field contract

本文件定义“导购小龙虾”相关飞书多维表格的**目标字段语义**，用于约束多个 open claw 写入同一张表时的字段含义一致性。

## 使用原则
- 本文件描述的是 **应有字段 / 目标 schema**，不是线上实时状态。
- 线上表里**实际已有字段**，必须在运行时通过飞书 `field-list` 读取，不依赖人工记忆。
- 写表前，先做：
  1. 自动发现候选表
  2. 读取真实字段列表
  3. 与本 contract 对比
  4. 输出：已存在字段 / 缺失字段 / 可疑字段 / 是否可直接写入
- 若字段名存在但写入语义不一致，以本 contract 为准修正 skill，不以历史脏数据为准。

---

# 1. 试衣商品记录（主表）

## 1.1 字段语义

| 字段名 | 含义 | 必需性 | 允许写入 | 禁止写入 |
|---|---|---|---|---|
| record_id | 单条试衣记录唯一标识 | required | 统一生成的记录 ID | 手机号、open_id、商品名 |
| guide_name | 导购展示姓名 | required | 张三、Luna | open_id、手机号 |
| guide_open_id | 导购飞书 open_id | recommended | `ou_xxx` | 姓名、手机号 |
| operator_id | 当前执行写入动作的操作者 / 实例标识 | recommended | open claw 实例 ID、员工内部 ID | 顾客手机号 |
| operator_name | 当前操作者展示名 | optional | 张三、小龙虾1号 | open_id |
| customer_id | 顾客统一唯一标识 | optional | 统一生成的 customer_uid | 导购 open_id、手机号直写 |
| customer_phone | 顾客手机号 | optional | 138xxxx | 导购 ID |
| member_mobile_last4 | 会员手机号后四位 | required | 1234 | 完整手机号、导购 ID |
| session_id | 单次 session 唯一标识 | required | session_xxx | 商品名、手机号 |
| store_name | 门店名称 | recommended | 杭州万象城店 | open_id |
| product_name | 商品名称 | required | 西装外套A12 | 历史上下文里的其他商品 |
| product_code | 商品编码 | recommended | A12 / SPU / SKU code | 导购 ID |
| sku | SKU | optional | sku_xxx | 导购 ID |
| fitting_date | 试衣日期 | required | `2026-04-29` | open_id |
| fitting_time | 试衣时间 | optional | `2026-04-29 10:20:00` | open_id |
| try_on_result | 试穿结果 | required | 合适/待考虑/未成交 | 纯图片猜测出的成交结论 |
| fit_feedback | 客户试穿反馈 | recommended | 太紧、喜欢版型 | open_id |
| body_effect_desc | 上身效果描述 | optional | 显瘦、肩线合适 | open_id |
| liked_points | 喜欢点 | optional | 面料舒服、版型挺括 | open_id |
| disliked_points | 不喜欢点 | optional | 颜色偏深 | open_id |
| not_buy_reason | 未购买原因 | optional | 预算卡点、等活动 | open_id |
| followup_intent | 是否有后续回访意向 | optional | 到货通知我 | open_id |
| source_channel | 来源渠道 | recommended | openclaw / feishu | 姓名 |
| source_sender_id | 原始消息发送人 ID | recommended | 飞书/IM sender id | 姓名 |
| source_chat_id | 原始聊天 ID | recommended | chat_xxx | 姓名 |
| source_agent_id | 当前写入的 open claw 实例 ID | recommended | openclaw_store_01 | 顾客手机号 |

## 1.2 关键硬规则
- `guide_name` 只能写“姓名/展示名”，不能写 open_id。
- `guide_open_id` 才能承接飞书 open_id；若表里没有此字段，应提示补字段，不得塞进 `guide_name`。
- `customer_id` 在没有统一 customer_uid 生成规则前，可留空；不得把手机号或导购 open_id 塞进去。
- `customer_phone` 与 `member_mobile_last4` 语义不同，不能混用。
- `product_name` 必须优先取本次识别结果，不能被历史上下文商品覆盖。

## 1.3 最小可写字段集
若要判定“主表当前可安全写入”，至少应具备：
- `record_id`
- `guide_name`
- `member_mobile_last4`
- `session_id`
- `product_name`
- `fitting_date`
- `try_on_result`

---

# 2. 线索回访表

## 2.1 字段语义

| 字段名 | 含义 | 必需性 | 允许写入 | 禁止写入 |
|---|---|---|---|---|
| lead_id | 回访线索唯一标识 | required | lead_xxx | 手机号直写 |
| source_fitting_record_id | 来源试衣记录 ID | required | record_xxx | 商品名 |
| guide_name | 导购展示姓名 | required | 张三 | open_id |
| guide_open_id | 导购飞书 open_id | recommended | `ou_xxx` | 姓名 |
| member_mobile_last4 | 会员尾号 | required | 1234 | 完整手机号 |
| product_name | 试穿单品 | required | 西装外套A12 | open_id |
| lead_reason | 回访原因 | required | 喜欢但未成交 | open_id |
| touch_time_type | 建议触达时机类型 | required | same_day / next_day / event_triggered | 任意自由文本 |
| planned_touch_at | 计划触达时间 | optional | `2026-04-29 18:00:00` | open_id |
| touch_trigger_condition | 触发条件 | optional | 到货/补码/活动 | open_id |
| followup_content | 建议回访内容 | optional | 新到同版浅色可试 | open_id |
| recommended_action | 推荐动作 | optional | 微信回访 / 电话回访 | open_id |
| priority | 优先级 | required | high / medium / low | open_id |
| status | 状态 | required | pending / waiting_trigger / cancelled | 任意混乱状态值 |
| remind_at | 提醒时间 | optional | `2026-04-29 17:00:00` | open_id |
| is_reminded | 是否已提醒 | optional | true / false | 文本“是/否” |

## 2.2 最小可写字段集
- `lead_id`
- `source_fitting_record_id`
- `guide_name`
- `member_mobile_last4`
- `product_name`
- `lead_reason`
- `touch_time_type`
- `priority`
- `status`

---

# 3. 会员画像表

## 3.1 字段语义

| 字段名 | 含义 | 必需性 | 允许写入 | 禁止写入 |
|---|---|---|---|---|
| profile_id | 画像记录唯一标识 | optional | profile_xxx | 手机号直写 |
| member_mobile_last4 | 会员尾号 | required | 1234 | open_id |
| style_preference | 风格偏好 | recommended | 通勤、极简 | open_id |
| fit_preference | 版型偏好 | recommended | 修身、落肩 | open_id |
| size_preference | 尺码偏好 | optional | L / 38 | open_id |
| body_shape_notes | 体型备注 | optional | 肩宽、腰细 | open_id |
| color_preference | 颜色偏好 | optional | 浅灰、藏蓝 | open_id |
| material_preference | 面料偏好 | optional | 羊毛、棉麻 | open_id |
| price_sensitivity | 价格敏感度 | optional | 1500-3000 可接受 | open_id |
| common_not_buy_reasons | 常见未购原因 | optional | 预算卡点 | open_id |
| followup_trigger_preferences | 回访触发偏好 | optional | 到货提醒、活动提醒 | open_id |
| styling_needs | 搭配需求 | optional | 通勤、见客户 | open_id |
| last_profile_update_at | 最近更新时间 | recommended | `YYYY-MM-DD HH:mm:ss` | open_id |
| profile_source_record_id | 最近来源试衣记录 | recommended | record_xxx | 商品名 |
| profile_confidence | 画像置信度 | optional | high / medium / low | 任意无约束文本 |

## 3.2 最小可写字段集
- `member_mobile_last4`
- `last_profile_update_at`

---

# 4. 运行时字段检查规则

写表前，skill 必须先读取飞书真实字段，并输出以下四类结果：

## 4.1 existing_fields
当前表里已存在、且与 contract 一致的字段。

## 4.2 missing_required_fields
contract 里标记为 `required`，但当前表里不存在的字段。

## 4.3 missing_recommended_fields
contract 里标记为 `recommended`，但当前表里不存在的字段。

## 4.4 suspicious_fields
名字存在，但语义上高风险污染的字段。例如：
- `guide_name` 历史上被写入 open_id
- `customer_id` 历史上被写入手机号或导购 ID

## 4.5 can_write
判定逻辑：
- 缺 `required` 字段 → `can_write = false`
- 仅缺 `recommended` 字段 → `can_write = true`，但要提示建议补字段
- 存在 `suspicious_fields` → 可以先提示人工确认或先治理后写

---

# 5. 建议新增字段

当前最值得补的字段：
- `guide_open_id`
- `customer_phone`
- `source_agent_id`

原因：
- 避免把 open_id 塞进 `guide_name`
- 避免把手机号塞进 `customer_id`
- 让不同 open claw 的来源可追踪
