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
- `lobster-fitting-bitable-sync`：主表粒度调整为“**一件商品 + 一个客人 + 一个 session = 一条记录**”，主表名调整为 `试衣商品记录`，并补充 `试衣Session索引` 作为可选辅助表。
