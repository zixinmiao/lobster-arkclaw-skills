# AGENTS.md - lobster-arkclaw-skills

本仓库是一套可复用的导购试衣技能包，默认面向“多导购共用同一套飞书试衣台账”的场景。

## Shared Bitable Rules

### 默认模式
- 默认使用**项目级共享 Base**
- 同一项目下，所有导购共用同一套飞书多维表格
- 不按导购个人、会话、session 单独创建 Base

### 建表触发规则
1. 正式写表前，先检查是否已有可用 `bitable_binding`
2. 若已有 binding，则直接复用，不再创建新 Base
3. 若没有 binding，才允许执行 `lobster-fitting-bitable-bootstrap`
4. 建表成功后，必须立即把 binding 持久化为项目级单例，供后续所有导购复用

### 写表前置顺序
1. 完成 bundle / draft / merge 等上游整理
2. 判断 `write_decision.allow_write`
3. 若允许写入，先检查项目级 `bitable_binding`
4. 若缺失，再执行 `lobster-fitting-bitable-bootstrap`
5. 仅当 `binding_status = ready` 时，才进入 `lobster-fitting-bitable-sync` 和正式写表

### 主表区分字段
主表 `试衣商品记录` 默认应保留以下字段：
- `guide_name`
- `store_name`
- `operator_id`
- `operator_name`
- `source_chat_id`
- `source_sender_id`
- `session_id`
- `brand_name`（跨品牌复用时推荐）

### 禁止事项
- 禁止因不同导购触发而重复创建多个 Base
- 禁止把 binding 仅保存在单次会话上下文中
- 禁止在已有可用 binding 时重复执行建表
- 禁止省略导购/录入来源字段，导致多人共用时无法区分记录归属
