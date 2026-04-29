---
name: lobster-fitting-bitable-sync
description: 将试衣合并结果映射成适合飞书多维表格写入的结构。适用于需要把“单客单商品单次试衣”记录 upsert 到飞书多维表格台账、补齐会员字段、保留 session 归属与素材映射、生成写入摘要时使用。
---

# 试衣飞书多维表格沉淀

把结构化试衣数据整理成适合飞书多维表格写入的结果。

## 依赖
- 当任务涉及飞书多维表格字段设计、记录写入、视图或公式语义时，优先配合 `lark-base` 类能力执行。
- 若当前缺少 `bitable_binding`、Base 或目标表结构，先使用 `lobster-fitting-bitable-bootstrap` 补齐基础设施，再生成正式写表载荷。
- 当前默认不依赖预置 `config.json`。优先通过“人工给链接，首次自动登记 binding”获取 binding。
- 写表前必须先读取 `../references/bitable-field-contract.md`，按 contract 判断字段语义、最小可写字段集和可疑字段。

## 输入
- `input_bundle`
- `session_link`
- `fitting_records`
- `media_records`
- `write_decision`
- `bitable_binding`
- `instance_context`（可选；用于补 `store_name / operator_id` 等来源字段）

## 链接初始化机制（主方案）
如果当前没有可用 `bitable_binding`，优先走“给链接 -> 自动登记”：

### 1. 接收人工提供的链接
优先接收以下任一链接：
- Base 链接
- 指向具体表的 Base 链接（带 `table=`）
- 带 `view=` 的表视图链接

### 2. 直接解析 binding
从链接中直接提取：
- `base_token`
- `table_id`
- `view_id`（若存在）

### 3. 读取真实字段并校验
解析后，不直接认定可写，必须读取字段结构并和 contract 比对。

当前试衣主表的高优先级识别字段：
- `member_mobile_last4`
- `product_name`
- `try_on_result`
- `guide_name`
- `fitting_date`
- `session_id`
- `product_code`
- `store_name`

### 4. 首次登记 binding
若链接可访问且字段检查通过：
- 输出解析结果
- 持久化保存为项目级 binding
- 后续所有 open claw 直接复用

## 自动发现机制（仅 fallback）
只有在“没有 binding、也没有人工提供链接”时，才允许尝试搜索候选表。
搜索命中后仍需人工确认，且不作为默认主路径。


## 字段检查流程
发现候选表后，不要直接写，先做字段检查：
1. 读取候选表真实字段列表
2. 对照 `../references/bitable-field-contract.md` 找出：
   - `existing_fields`
   - `missing_required_fields`
   - `missing_recommended_fields`
   - `suspicious_fields`
3. 只有 `missing_required_fields = []` 时，才允许进入正式写入
4. 如果发现 `guide_name`、`customer_id` 等字段存在历史污染，应优先提示治理或改写到新增字段，而不是继续混写

## 数据粒度
### 主表粒度
主表 `试衣商品记录` 的一条记录定义为：

> 一件商品 + 一个客人 + 一个 session = 一条记录

若同一 session 中试了 3 件商品，应写入 3 条主记录。

## 字段映射
### fitting_record -> 试衣商品记录（目标字段语义）
优先映射以下字段语义；真实字段名以自动发现后的候选表结构为准：
- `record_id`
- `customer_id`
- `member_mobile_last4`
- `liked_points`
- `session_id`
- `session_status`
- `source_channel`
- `guide_name`
- `source_sender_id`
- `fit_feedback`
- `try_on_result`
- `source_bundle_id`
- `ocr_confidence`
- `raw_notes`
- `size`
- `body_effect_desc`
- `source_chat_id`
- `tag_price`
- `not_buy_reason`
- `operator_id`
- `product_name`
- `is_member`
- `source_message_ids`
- `color`
- `followup_intent`
- `store_name`
- `member_identity_status`
- `disliked_points`
- `is_high_intent`
- `fitting_time`
- `sku`
- `operator_name`
- `fitting_date`
- `product_code`

### 来源字段兜底规则
如果实例侧还没有门店配置，不要阻塞写表准备：
- `store_name` 默认写 `unknown`
- `source_channel` 默认写 `openclaw`
- 若链路里能拿到实例标识，优先写入 `operator_id / operator_name / source_sender_id`
- 当前表里若没有独立 `source_agent_id / source_store_id` 字段时，不强依赖；先依靠 `operator_id / source_chat_id / source_sender_id / session_id` 做追踪

### 可扩展关联表
以下表由其他 skill 负责生成或更新，但应与试衣主表保留可关联字段：
- `线索回访表`（由 `lobster-followup-lead-sync` 负责）
- `会员画像表`（由 `lobster-member-profile-sync` 负责）

### 商品主值映射约束
- 多维表格中的商品主字段，应直接映射自当前 `fitting_record.product_name / product_code / sku`。
- 当本次输入包含新的吊牌 OCR 结果时，写表层不得再回退使用历史上下文商品名覆盖当前主值。
- 若上游 `merge_notes` 已标记商品冲突，写表层应优先保留本次 bundle 的商品主值，并把冲突说明保留到摘要字段，而不是把旧商品直接写进主表。

## dedupe 建议
### 主表 dedupe_key
优先使用：
- `record_id`

若上游未显式生成 `record_id`，退化为：
- `session_id + customer_id + product_code`

## 输出目标
输出 `sync_result`：
- `tables`: 各表待写入记录
- `write_mode`: `upsert`
- `dedupe_keys`
- `sync_status`
- `sync_summary`

## 执行规则
- 缺 binding 时，不要伪造一个固定表配置；应先进入“人工给链接 -> 自动登记 binding”流程。
- 若既没有 binding，也没有可用链接，`sync_status` 应标记为 `needs_confirmation` 或 `no_binding`。
- 缺关键字段时，标记 `partial` 并在 `sync_summary` 说明。
- 若只拿到 `session` 级摘要、尚未拆成 `fitting_records`，应优先提示上游先完成记录粒度转换，不要擅自按旧结构落表。
- 若 `session_link.link_action = pending_confirm`，允许输出待写入载荷，但应在 `sync_status` 或 `sync_summary` 中明确说明当前归属仍待确认。
- 若 `write_decision.allow_write = false`，则连接器/写入器必须停止向主表写入；其中当 `write_decision.write_block_reason = PENDING_MEDIA_NOT_READY_TO_WRITE` 时，应直接跳过正式记录创建。
- 若 `input_bundle.bundle_status = pending_media` 且 `fitting_records = []`，则不得向主表 `试衣商品记录` 写入任何正式记录；此时仅允许输出待处理信息。
- 若上游只提供单张图片且无明确文本，不得在 `sync_summary` 或任何落表字段中推断“成交/已购买/已完成”。

## 输出格式
返回：
```json
{"sync_result": {
  "tables": {
    "试衣商品记录": []
  },
  "write_mode": "upsert",
  "dedupe_keys": {
    "试衣商品记录": ["record_id"]
  },
  "sync_status": "ready | partial | pending_confirm | needs_confirmation | no_binding",
  "sync_summary": "..."
}}
```
