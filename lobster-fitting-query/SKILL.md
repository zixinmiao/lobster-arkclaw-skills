---
name: lobster-fitting-query
description: 查询并汇总试衣主记录与商品明细。适用于用户按日期、门店、导购、会员尾号、SKU、款号、颜色、尺码、是否会员等条件查询试衣记录，或需要生成明细加统计摘要时使用。
---

# 试衣单查询与统计

根据查询条件生成查询计划、结果结构和摘要。

## 支持过滤条件
- `date`
- `date_range`
- `store_name`
- `guide_name`
- `member_mobile_last4`
- `sku`
- `product_code`
- `color`
- `size`
- `is_member`

## 查询模式
- `detail`
- `summary`
- `detail_and_summary`（默认）

## 输出目标
输出：
- `records`
- `summary`
- `summary_text`
- `query_status`
- `query_plan`

### summary
优先包含：
- `session_count`
- `item_count`
- `member_session_count`
- `guide_ranking`
- `sku_ranking`

## 执行规则
- 若有真实表连接，则按连接执行。
- 若当前只拿到数据样本或自然语言条件，则先生成标准查询条件与期望结果结构。
- 结果为空时，明确返回空结果，不强行总结。

## 输出格式
返回：
```json
{
  "records": [],
  "summary": {},
  "summary_text": "",
  "query_status": "success | empty | missing_binding",
  "query_plan": {}
}
```
