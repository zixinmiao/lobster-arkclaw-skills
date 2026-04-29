---
name: lobster-fitting-daily-summary
description: 汇总某日试衣录入情况并生成可直接发送的日报。适用于门店晚间复盘、店长查看当日录入单量、商品件数、会员单量、异常记录和导购表现时使用。
---

# 试衣日报汇总

生成可直接发送的试衣日报。

## 默认统计项
- `fitting_session_count`
- `fitting_item_count`
- `member_session_count`
- `abnormal_record_count`
- `top_guides`
- `top_products`

## 异常规则
- `missing_product_code`
- `missing_size`
- `low_ocr_confidence`

## 输出目标
输出：
- `summary_result`
- `summary_text`
- `summary_status`

### summary_text
用业务可转发口径输出，优先包括：
- 日期
- 录入单量
- 商品件数
- 会员试衣单数
- 异常记录数
- 表现亮点
- 待补录提醒

## 执行规则
- 未指定日期时默认当天。
- 有数据源就汇总真实数据；没有数据源就输出汇总模板和缺失项。
- 不夸大亮点，不隐去异常。

## 输出格式
返回：
```json
{"summary_result": {...}, "summary_text": "...", "summary_status": "success | empty | missing_binding"}
```
