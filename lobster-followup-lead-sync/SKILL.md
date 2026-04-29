---
name: lobster-followup-lead-sync
description: 根据试衣反馈判断是否生成回访线索，并同步写入线索回访表。适用于需要稳定落回访必填字段的场景。
---

# 试衣回访线索生成与落表

## 目标
基于单条试衣记录，判断是否需要生成回访线索，并输出可直接写入“线索回访表”的结构化结果。

## 字段 contract
写表前必须先读取 `../references/bitable-field-contract.md`，确认当前目标字段语义、最小可写字段集和可疑字段。

## 链接初始化机制
执行前优先使用人工提供链接做首次登记：
- 先确认是否已有已保存的 `bitable_binding`
- 若无，则优先接收对应表链接并直接解析 `base_token / table_id / view_id`
- 再读取真实字段结构，确认目标表可用
- 只有在没有 binding、也没有链接时，才 fallback 搜索候选表

## 输入
- `fitting_record`
- `member_context`（可选）
- `store_context`（可选）
- `current_time`（可选）
- `campaign_context`（可选）

## 输出目标
返回 `result`：
- `followup_lead`
- `decision_reason`

### followup_lead
当前回访表只保留并写入以下字段：
- `线索ID`
- `来源试衣记录ID`
- `日期`
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

### 必填字段强校验
正式写入前，以下字段必须全部非空：
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

若任一字段缺失：
- 不得正式写入回访表
- 应输出缺失字段说明
- 应把当前结果标记为 `blocked` / `partial`

## 生成规则
- 只有“有兴趣 + 未成交 + 原因明确 + 后续可被导购动作解决”时，优先生成线索。
- `来源试衣记录ID` 应直接引用主表 `session_id`。
- `导购名称` 来自 `guide_name`。
- `导购open_id` 应来自 `operator_id` 或稳定的导购 open_id 映射，不得留空。
- `试穿单品` 应优先拼接 `product_name + product_code`。
- `是否已提醒` 创建时默认写 `false`。
- `状态` 创建时默认优先写 `pending`；若是触发式线索可写 `waiting_trigger`。

## 输出格式
```json
{"result": {...}}
```
