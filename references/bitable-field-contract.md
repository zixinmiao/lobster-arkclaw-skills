# bitable field contract

本文件定义“导购小龙虾”相关飞书多维表格的目标字段语义，用于保证不同 open claw 在写表时对必填字段有一致理解，并在写入前做强制校验。

## 使用原则
- 本文件描述的是应有字段 / 目标 schema，不是线上实时状态。
- 线上表里实际已有字段，必须在运行时通过飞书 `field-list` 读取。
- 写表前必须先做字段检查，判断：`existing_fields` / `missing_required_fields` / `missing_recommended_fields` / `suspicious_fields`。
- 只要 `missing_required_fields` 非空，就不得正式写入。

---

# 1. 试衣商品记录（主表）

## 1.1 字段语义

| 字段名 | 含义 | 必需性 | 允许写入 | 禁止写入 |
|---|---|---|---|---|
| session_id | 单条试衣记录唯一标识 | required | 统一生成的记录 ID | 手机号、open_id、商品名 |
| guide_name | 导购展示姓名 | required | 张三、Luna | open_id、手机号 |
| session_status | 当前记录状态 | optional | open / closed / pending_confirm | 编造成交结论 |
| fitting_date | 试衣日期 | required | `2026/4/29` | open_id |
| fitting_time | 试衣时间 | required | `2026/4/29 10:20` | open_id |
| store_name | 门店名称 | optional | 杭州万象城店 | open_id |
| operator_id | 当前执行写入动作的操作者 / 实例标识 / 飞书 open_id | required | open claw 实例 ID、员工内部 ID、飞书 open_id | 顾客手机号 |
| is_member | 布尔值；识别到会员则为 1，否则为 0 | required | `true` / `false` | 任意文本 |
| member_mobile_last4 | 识别到会员手机号的后四位数字 | optional | 0832 | 非数字文本 |
| product_code | 商品货号 / 款号 | required | `EBF2JEN027-S97` | 导购 ID |
| product_name | 商品名称 | required | `MO&Co.牛仔裤` | 历史上下文里的其他商品 |
| color | 商品颜色 | optional | 牛仔蓝色 | open_id |
| tag_price | 商品价格 | optional | 1699 | 非价格文本 |
| ocr_confidence | OCR 识别准确性 | optional | `0~1` 数值 | 0~1 之外的数值 |
| try_on_result | 试穿结果 | required | 合适 / 待考虑 / 未成交 | 纯图片猜测出的成交结论 |
| body_effect_desc | 客户试穿反馈 | recommended | 太紧、喜欢版型 | open_id |
| fit_feedback | 上身效果描述 | optional | 显瘦、肩线合适 | open_id |
| liked_points | 喜欢点 | optional | 面料舒服、版型挺括 | open_id |
| disliked_points | 不喜欢点 | optional | 颜色偏深 | open_id |
| followup_intent | 是否有后续回访意向 | optional | 到货通知我 | open_id |
| source_channel | 来源渠道 | recommended | openclaw / feishu | 姓名 |
| source_bundle_id | 原始消息发送人 ID（飞书场景下应写 open_id） | required | 飞书 open_id / IM sender id | 姓名、手机号 |
| source_message_ids | 原始聊天 / 消息标识 | recommended | chat_xxx / msg_xxx | 姓名 |
| raw_notes | 原始文字信息 | required | 原始试衣描述文本 | 空值 |
| size | 商品尺码 | optional | `165/66A(26)` | open_id |
| not_buy_reason | 最终没有购买的阻碍原因 | optional | 有同款 / 预算卡点 | open_id |

## 1.2 关键硬规则
- `guide_name` 只能写姓名/展示名，不能写 open_id。
- `session_id` 是主表唯一标识字段，写表时必须稳定生成。
- `source_bundle_id` 在飞书场景下必须直接透传 sender open_id，不得留空。
- `operator_id` 必须稳定填写，优先使用实例标识或飞书 open_id。
- `is_member = true` 时，若已识别出会员手机号或尾号，应优先回填 `member_mobile_last4`。
- `product_name` / `product_code` 必须优先取本次识别结果，不能被历史上下文商品覆盖。

## 1.3 最小可写字段集
- `session_id`
- `guide_name`
- `fitting_date`
- `fitting_time`
- `operator_id`
- `is_member`
- `product_code`
- `product_name`
- `try_on_result`
- `source_bundle_id`
- `raw_notes`

---

# 2. 线索回访表

## 2.1 字段语义

| 字段名 | 含义 | 必需性 | 允许写入 | 禁止写入 |
|---|---|---|---|---|
| 线索ID | 回访线索唯一标识 | required | lead_xxx | 手机号直写 |
| 来源试衣记录ID | 来源试衣记录 ID | required | session_xxx | 商品名 |
| 日期 | 线索生成日期 | required | `2026-04-28` | open_id |
| 导购名称 | 导购展示姓名 | required | 子心 | open_id |
| 导购open_id | 导购飞书 open_id | required | `ou_xxx` | 姓名 |
| 会员尾号 | 会员尾号 | required | 2363 | 完整手机号 |
| 试穿单品 | 试穿单品 | required | `MO&Co.牛仔裤 EBF2JEN027-S97` | open_id |
| 回访原因 | 回访原因 | required | 高意向，考虑中，适时跟进 | open_id |
| 建议触达时机类型 | 建议触达时机类型 | required | next_day | 任意自由文本 |
| 计划触达时间 | 计划触达时间 | optional | `2026/4/1 18:00` | open_id |
| 建议回访内容 | 建议回访内容 | required | 回访建议文案 | open_id |
| 推荐动作 | 推荐动作 | required | 主动联系询问考虑进度 | open_id |
| 优先级 | 优先级 | required | 高 / 中 / 低 | open_id |
| 状态 | 状态 | required | pending | 任意混乱状态值 |
| 是否已提醒 | 是否已提醒 | required | true / false | 文本“是/否” |
| 提醒时间 | 提醒时间 | optional | `2026/4/1 17:00` | open_id |

## 2.2 最小可写字段集
- `线索ID`
- `来源试衣记录ID`
- `日期`
- `导购名称`
- `导购open_id`
- `会员尾号`
- `试穿单品`
- `回访原因`
- `建议触达时机类型`
- `建议回访内容`
- `推荐动作`
- `优先级`
- `状态`
- `是否已提醒`

---

# 3. 会员画像表

## 3.1 字段语义

| 字段名 | 含义 | 必需性 | 允许写入 | 禁止写入 |
|---|---|---|---|---|
| 会员尾号 | 会员手机号的后四位数字 | required | 2291 | open_id |
| 风格偏好 | 风格偏好 | optional | 避免宽松/慵懒/邋遢风格；倾向利落、精神、有型、线条感强 | open_id |
| 版型偏好 | 版型偏好 | optional | 合体/修身剪裁；有明显肩线和腰线；拒绝 oversize / 宽松慵懒款 | open_id |
| 尺码偏好 | 尺码偏好 | optional | S码（155/80A） | open_id |
| 颜色偏好 | 颜色偏好 | optional | 待补充（藏青色上身效果未满意，可继续探索其他色） | open_id |
| 面料偏好 | 面料偏好 | optional | 羊毛、棉麻 | open_id |
| 价格敏感度 | 价格敏感度 | optional | 可接受 1500-3000 | open_id |
| 常见未购原因 | 常见未购原因 | optional | 上身效果不满意（风格不符） | open_id |
| 搭配需求 | 搭配需求 | optional | 推荐有肩线、收腰、合体剪裁单品 | open_id |
| 体型备注 | 体型备注 | optional | 155/80A（S） | open_id |
| 最近更新时间 | 最近更新时间 | required | `2026-04-28` | open_id |
| 最近来源试衣记录 | 来源试衣记录中的 sessionID | required | session_xxx | 商品名 |
| 画像置信度 | 画像置信度 | required | 中（单次样本，建议持续补充） | 任意无约束文本 |

## 3.2 最小可写字段集
- `会员尾号`
- `最近更新时间`
- `最近来源试衣记录`
- `画像置信度`

---

# 4. 运行时字段检查规则

写表前，skill 必须先读取飞书真实字段，并输出以下四类结果：
- `existing_fields`
- `missing_required_fields`
- `missing_recommended_fields`
- `suspicious_fields`

## can_write 判定
- 缺 `required` 字段 → `can_write = false`
- 仅缺 `recommended` 字段 → `can_write = true`，但要提示建议补字段
- 存在 `suspicious_fields` → 先人工确认或先治理后写

---

# 5. 当前收敛原则

当前优先目标不是继续扩字段，而是：
- 按当前保留字段完成稳定写入
- 对 required 字段做强制校验
- 减少表结构与 skill 口径之间的不一致
