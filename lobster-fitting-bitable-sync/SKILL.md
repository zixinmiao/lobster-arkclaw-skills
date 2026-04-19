---
name: lobster-fitting-bitable-sync
description: 将试衣主记录、商品明细和素材索引映射成适合飞书多维表格写入的结构。适用于需要把一次试衣数据 upsert 到飞书多维表格台账、补齐会员字段、生成写入摘要时使用。
---

# 试衣飞书多维表格沉淀

把结构化试衣数据整理成适合飞书多维表格写入的结果。

## 依赖
当任务涉及飞书多维表格字段设计、记录写入、视图或公式语义时，优先配合 `lark-base` 类能力执行。

## 输入
- `session_record`
- `item_records`
- `media_records`
- `bitable_binding`

## 目标表
- `试衣主记录`
- `试衣商品明细`
- `试衣素材索引`

## 字段映射
### session
- `session_id`
- `fitting_date`
- `fitting_time`
- `guide_name`
- `store_name`
- `is_member`
- `member_mobile_last4`
- `member_identity_status`
- `notes`
- `source_channel`
- `source_message_id`

### item
- `session_id`
- `sku`
- `product_code`
- `product_name`
- `color`
- `size`
- `tag_price`
- `ocr_confidence`

### media
- `session_id`
- `media_id`
- `media_type`
- `media_url`
- `media_source`

## 输出目标
输出 `sync_result`：
- `tables`: 各表待写入记录
- `write_mode`: `upsert`
- `dedupe_keys`: `session_id` `product_code` `media_id`
- `sync_status`
- `sync_summary`

## 执行规则
- 缺 binding 时，先输出待写入载荷，不伪造写入成功。
- 缺关键字段时，标记 `partial` 并在 `sync_summary` 说明。
- 输出应便于后续由连接器或脚本直接执行。

## 输出格式
返回：
```json
{"sync_result": {...}}
```
