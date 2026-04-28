# WORKFLOW.md - lobster-arkclaw-skills

本文档用于定义这套 skill 的推荐编排顺序。

> 说明：`WORKFLOW.md` 负责描述“应该如何串联”，不会自己执行。真正执行仍需要 ArkClaw / OpenClaw 的 runtime、connector、workflow engine 或上层服务来调用这些 skill。

## 目标

这套技能包默认面向以下场景：

- 多导购共用同一套飞书试衣台账
- 首次使用时自动创建飞书多维表格
- 表格创建成功后，后续导购复用已有表格
- 试衣记录、素材记录、session 索引写入同一套项目级 Base

## 总体编排原则

1. 先归并，再识别，再合并，再判断是否允许写入
2. 正式写表前，必须先确保 `bitable_binding` 已存在且可用
3. 若没有 binding，先执行建表 bootstrap；若已有 binding，则直接复用
4. `pending_media` 或 `write_decision.allow_write = false` 时，不得写入主表
5. 多导购共用一套 Base，禁止因不同导购触发而重复创建 Base

## 主流程

```text
message input
-> lobster-fitting-bundle-router
-> lobster-fitting-draft-manager
-> lobster-tag-ocr / lobster-fitting-stt / lobster-member-identity-parse
-> lobster-fitting-record-merge
-> [optional] lobster-fitting-card-fill
-> lobster-fitting-bitable-bootstrap
-> lobster-fitting-bitable-sync
-> base writer
-> lobster-followup-lead-sync
-> lobster-member-profile-sync
```

## 详细流程

### 1. 输入归并
先执行：
- `lobster-fitting-bundle-router`

目标：
- 把同一导购在短时间窗口内发送的图 / 音 / 文归并到同一个 bundle
- 避免一条消息直接变成一条正式记录

输出重点：
- `bundle_id`
- `bundle_status`
- `message_ids`
- `routing_action`

### 2. 草稿管理
再执行：
- `lobster-fitting-draft-manager`

目标：
- 管理草稿态
- 决定当前是否继续等待补充内容
- 在正式写入前先锁住“未确认”状态

输出重点：
- `draft_status`
- `allow_write`
- `missing_fields`

### 3. 识别层
按需执行：
- `lobster-tag-ocr`
- `lobster-fitting-stt`
- `lobster-member-identity-parse`

目标：
- 识别商品候选信息
- 转写语音
- 识别顾客身份和会员相关信息

### 4. 记录合并
执行：
- `lobster-fitting-record-merge`

目标：
- 形成结构化 `fitting_records`
- 判断 `session_link`
- 输出 `media_records`
- 生成 `write_decision`

关键约束：
- 如果只有图片、没有足够文本，必须返回 `pending_media`
- 不得仅依据单张图片推断成交
- 若当前 OCR 商品与历史上下文冲突，优先使用本次 OCR 商品

### 5. 可选卡片确认
若业务需要人工确认，再执行：
- `lobster-fitting-card-fill`

适用场景：
- 试衣信息还需导购二次确认
- 需要卡片预填并提交后再正式写表

### 6. 建表 / 绑定修复
正式写表前必须执行：
- `lobster-fitting-bitable-bootstrap`

规则：
- 若不存在 `bitable_binding.base_token`，先创建 Base 和目标表
- 若已有 binding，但目标表缺失，则补建缺表
- 若表已存在但字段不完整，则补字段
- 若已有可用项目级 binding，则直接复用，禁止重复创建 Base

默认创建对象：
- Base：`导购小龙虾试衣台账`
- 主表：`试衣商品记录`
- 辅助表：`试衣素材索引`
- 辅助表：`试衣Session索引`

### 7. 写表载荷生成
执行：
- `lobster-fitting-bitable-sync`

目标：
- 把结构化试衣数据映射成适合飞书 Base 写入的 payload
- 保留主表、素材表、session 索引表的写入结构

主表必备区分字段：
- `guide_name`
- `store_name`
- `operator_id`
- `operator_name`
- `source_chat_id`
- `source_sender_id`
- `session_id`

### 8. 正式写入
由 runtime / connector / workflow engine 执行实际 Base 写入。

写入前必须满足：
- `binding_status = ready`
- `write_decision.allow_write = true`
- 主表结构完整

禁止写入场景：
- `pending_media`
- `write_decision.allow_write = false`
- binding 不可用

### 9. 后置沉淀
成功写表后，可继续执行：
- `lobster-followup-lead-sync`
- `lobster-member-profile-sync`
- `lobster-fitting-daily-summary`

目标：
- 生成回访线索
- 更新会员画像
- 汇总日报

## 最小可用流程

如果当前只想先跑通“自动建表 + 写表”，可使用最小流程：

```text
message input
-> lobster-tag-ocr / lobster-fitting-stt
-> lobster-fitting-record-merge
-> lobster-fitting-bitable-bootstrap
-> lobster-fitting-bitable-sync
-> base writer
```

## 推荐前置判断

推荐在 runtime / connector 层固定加上这段逻辑：

```text
if no project-level bitable_binding:
  run lobster-fitting-bitable-bootstrap
else:
  reuse existing binding

if binding_status != ready:
  stop write

if write_decision.allow_write != true:
  stop write

run lobster-fitting-bitable-sync
run base writer
```

## 不应由 WORKFLOW.md 代替的部分

`WORKFLOW.md` 不代替以下内容：
- skill 自身的字段约束
- runtime 的实际调用逻辑
- connector 的鉴权与重试机制
- 飞书 Base 的真实写入命令

这些仍需由：
- `SKILL.md`
- `AGENTS.md`
- 上层 connector / workflow engine
共同完成。
