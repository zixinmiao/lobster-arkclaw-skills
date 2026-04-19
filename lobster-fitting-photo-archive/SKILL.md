---
name: lobster-fitting-photo-archive
description: 整理一次试衣场景中的图片、吊牌图、试衣图和语音附件索引。适用于需要把素材和试衣主记录关联起来，形成可追溯媒体台账时使用。
---

# 试衣素材归档

整理媒体资产索引，不执行真实上传。

## 输出目标
输出 `result`：
- `media_records`: 数组
- `archive_status`
- `archive_notes`

每条 `media_records` 尽量包含：
- `session_id`
- `media_id`
- `media_type`: `tag_image` | `fitting_image` | `voice` | `other`
- `media_url`
- `media_source`
- `capture_time`

## 执行规则
- 本 skill 负责结构化归档描述，不假设真实对象存储已成功写入。
- 没有 URL 时，保留原始附件标识或来源描述。
- 无法判断类型时标为 `other` 并备注。

## 输出格式
返回：
```json
{"result": {...}}
```
