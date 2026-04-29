---
name: lobster-fitting-bitable-sync
description: 将试衣合并结果映射成适合飞书多维表格写入的结构。适用于需要把试衣记录稳定写入飞书主表，并对必填字段做强制校验的场景。
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
1. 接收 Base / Table / View 链接
2. 解析 `base_token / table_id / view_id`
3. 读取真实字段并做 contract 校验
4. 校验通过后登记项目级 binding

## 字段检查流程
写表前不要直接落表，先做字段检查：
1. 读取候选表真实字段列表
2. 对照 `../references/bitable-field-contract.md` 找出：
   - `existing_fields`
   - `missing_required_fields`
   - `missing_recommended_fields`
   - `suspicious_fields`
3. 只有 `missing_required_fields = []` 时，才允许进入正式写入
4. 发现历史污染时，应优先提示治理，不要继续混写

## 主表字段映射
### fitting_record -> 试衣商品记录
当前主表只保留并写入以下字段：
- `session_id`
- `guide_name`
- `session_status`
- `fitting_date`
- `fitting_time`
- `store_name`
- `operator_id`
- `is_member`
- `member_mobile_last4`
- `product_code`
- `product_name`
- `color`
- `tag_price`
- `ocr_confidence`
- `try_on_result`
- `body_effect_desc`
- `fit_feedback`
- `liked_points`
- `disliked_points`
- `followup_intent`
- `source_channel`
- `source_bundle_id`
- `source_message_ids`
- `raw_notes`
- `size`
- `not_buy_reason`

### 必填字段强校验
正式写表前，以下字段必须全部非空：
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

若任一字段缺失：
- `write_mode` 不得进入正式写入
- `sync_status` 应标记为 `partial` 或 `blocked`
- `sync_summary` 中必须明确列出缺失字段

### 关键写入规则
- `session_id` 是主表唯一标识；不再依赖 `record_id`。
- `source_bundle_id` 在飞书场景下应直接透传 sender open_id。
- `is_member` 必须显式写 `true / false`，不能省略。
- `member_mobile_last4` 可选，但当 `is_member = true` 且已识别到尾号时应尽量回填。
- `product_name / product_code` 必须优先取本次识别结果，不能被历史上下文商品覆盖。
- `try_on_result` 必须来自文本/语音/确认信息，不得仅凭图片猜测。

## dedupe 建议
主表建议使用：
- `session_id`

## 输出目标
输出 `sync_result`：
- `tables`
- `write_mode`
- `dedupe_keys`
- `sync_status`
- `sync_summary`

## 执行规则
- 缺 binding 时，不要伪造固定表配置；应先进入“人工给链接 -> 自动登记 binding”流程。
- 若既没有 binding，也没有可用链接，`sync_status` 应标记为 `needs_confirmation` 或 `no_binding`。
- 若 `missing_required_fields` 非空，应阻断正式写入。
- 若只拿到 `session` 级摘要、尚未拆成可写主记录，不要擅自落表。
- 若 `write_decision.allow_write = false`，连接器必须停止向主表写入。
- 若 `input_bundle.bundle_status = pending_media` 且 `fitting_records = []`，不得向主表写入任何正式记录。

## 输出格式
返回：
```json
{"sync_result": {
  "tables": {
    "试衣商品记录": []
  },
  "write_mode": "upsert | blocked",
  "dedupe_keys": {
    "试衣商品记录": ["session_id"]
  },
  "sync_status": "ready | partial | blocked | pending_confirm | needs_confirmation | no_binding",
  "sync_summary": "..."
}}
```
