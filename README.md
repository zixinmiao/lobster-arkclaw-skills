# lobster-arkclaw-skills

导购试衣与飞书多维表格场景的 ArkClaw/OpenClaw 标准 Skill 集合。

## Skills
- lobster-fitting-stt
- lobster-tag-ocr
- lobster-member-identity-parse
- lobster-fitting-record-merge
- lobster-fitting-photo-archive
- lobster-fitting-bitable-sync
- lobster-fitting-query
- lobster-fitting-daily-summary
- lobster-script-generate

## Usage
将仓库作为自定义 skill 源，供 ArkClaw / OpenClaw 学习或安装。

## 最新调整
- `lobster-fitting-record-merge`：新增飞书拆分消息归并（bundle）、session 归属判断（append / new_session / pending_confirm）和跨天默认新建规则。
- `lobster-fitting-record-merge`：补充强约束——当只有单张图片且无文本时，`bundle_status` 必须为 `pending_media`，`fitting_records` 必须为空，且禁止仅依据图片推断成交。
- `lobster-fitting-record-merge`：新增硬字段 `write_decision`，其中 `PENDING_MEDIA_NOT_READY_TO_WRITE` 作为连接器可直接识别的禁止写入标记。
- `lobster-fitting-bitable-sync`：主表粒度调整为“**一件商品 + 一个客人 + 一个 session = 一条记录**”，主表名调整为 `试衣商品记录`，并补充 `试衣Session索引` 作为可选辅助表。
- `lobster-fitting-bitable-sync`：补充执行约束——`pending_media` 状态下不得写入主表正式记录，并要求连接器识别 `write_decision.allow_write = false`。
- `lobster-fitting-record-merge` / `lobster-fitting-bitable-sync`：补充商品主值优先级约束——本次吊牌 OCR 识别出的商品应优先于历史上下文商品，防止“试穿单品”写成旧商品。
