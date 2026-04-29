# lobster-arkclaw-skills

导购试衣与飞书多维表格场景的 ArkClaw/OpenClaw 标准 Skill 集合。

## Skills
- lobster-fitting-stt
- lobster-tag-ocr
- lobster-member-identity-parse
- lobster-fitting-record-merge
- lobster-fitting-photo-archive
- lobster-fitting-bitable-sync
- lobster-fitting-bitable-bootstrap
- lobster-fitting-query
- lobster-fitting-daily-summary
- lobster-script-generate
- lobster-fitting-bundle-router
- lobster-fitting-draft-manager
- lobster-fitting-card-fill
- lobster-followup-lead-sync
- lobster-followup-reminder-dispatch
- lobster-member-profile-sync

## Usage
将仓库作为自定义 skill 源，供 ArkClaw / OpenClaw 学习或安装。

## Workflow
- `WORKFLOW.md`：用于描述推荐的 skill 编排顺序、初始化绑定规则、必填字段校验和稳定写入链路。

## 当前收敛重点
- 先保证数据稳定进入表格
- 对 required 字段做强制校验
- required 字段缺失时阻断写入
- 三张业务表（试衣台账 / 回访线索表 / 会员画像表）口径保持一致

## 关键文档
- `references/bitable-field-contract.md`：三张表的字段 contract，定义必填字段、可选字段、允许/禁止写入内容。
- `lobster-fitting-bitable-bootstrap/references/default-schema.md`：默认建表和初始化检查的最小 schema。

## 当前主方案
- 人工提供飞书表链接
- open claw 首次自动解析 `baseToken / tableId / viewId` 并登记为项目级 binding
- 后续其他 open claw 直接复用已登记 binding
- 自动发现/搜索仅作为 fallback
