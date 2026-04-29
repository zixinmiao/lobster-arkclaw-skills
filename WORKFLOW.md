# WORKFLOW.md - lobster-arkclaw-skills

本文档用于定义这套 skill 的推荐编排顺序。

> 说明：`WORKFLOW.md` 负责描述应该如何串联；真正执行仍需要 runtime、connector 或 workflow engine 来调用这些 skill。

## 目标

这套技能包默认面向以下场景：
- 多导购共用同一套飞书试衣台账
- 首次使用时完成飞书表绑定或初始化
- 试衣记录、回访线索、会员画像稳定写入同一套项目级数据底座
- required 字段缺失时阻断写入，优先保证数据稳定性

## 总体编排原则
1. 先归并，再识别，再合并，再判断是否允许写入
2. 正式写表前，必须先确保 `bitable_binding` 已存在且可用
3. 若没有 binding，优先走“给链接 -> 自动登记”
4. `pending_media` 或 `write_decision.allow_write = false` 时，不得写入主表
5. required 字段缺失时，不得部分落表

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

## 关键前置规则
### 1. 导购资料继承
runtime / connector 应先读取 sender 级资料缓存，至少尝试继承：
- `guide_name`
- `store_name`
- `operator_id`
- `source_bundle_id`

若同一 sender 之前已经明确提供过导购姓名或门店，后续消息默认继承，不应反复询问。

### 2. 主表 required 字段
主表写入前必须具备：
- `session_id`
- `guide_name`
- `fitting_date`
- `fitting_time`
- `operator_id`
- `is_member`
- `product_code`
- `product_name`
- `try_on_result`
- `source_bundle_id`
- `raw_notes`

### 3. source_bundle_id 透传规则
如果来源是飞书消息，runtime / connector 必须把 sender open_id 作为 `source_bundle_id` 直接透传到后续链路。
该字段不依赖模型提取，不依赖导购补充。

### 4. 语音处理边界
当前若没有可用 transcript，不应假设可以直接把录音转成结构化字段；无 transcript 时应进入待补录 / 待补文本状态。

## 建表 / 初始化规则
正式写表前必须执行：
- `lobster-fitting-bitable-bootstrap`

默认创建 / 校验对象：
- Base：`导购小龙虾试衣台账`
- `试衣商品记录`
- `试衣素材索引`
- `试衣Session索引`
- `线索回访表`
- `会员画像表`

初始化检查时，不仅检查表是否存在，也要检查 required 字段是否完整。

## 写表载荷生成
执行：
- `lobster-fitting-bitable-sync`

目标：
- 把结构化试衣数据映射成适合飞书 Base 写入的 payload
- 在正式写入前完成 required 字段校验

## 正式写入前必须满足
- `binding_status = ready`
- `write_decision.allow_write = true`
- `missing_required_fields = []`
- 主表结构完整

禁止写入场景：
- `pending_media`
- `write_decision.allow_write = false`
- binding 不可用
- required 字段缺失

## 后置沉淀
成功写表后，按条件继续执行：
- `lobster-followup-lead-sync`
- `lobster-member-profile-sync`
- `lobster-fitting-daily-summary`

### 回访表 required 字段
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

### 会员画像表 required 字段
- `会员尾号`
- `最近更新时间`
- `最近来源试衣记录`
- `画像置信度`

## 推荐前置判断

```text
if no project-level bitable_binding:
  if table/base link is provided:
    parse link -> validate fields -> save binding
  else:
    run bootstrap fallback

if binding_status != ready:
  stop write

if write_decision.allow_write != true:
  stop write

if missing_required_fields is not empty:
  stop write

run lobster-fitting-bitable-sync
run base writer
```
