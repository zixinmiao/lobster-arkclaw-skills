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
- `WORKFLOW.md`：用于描述推荐的 skill 编排顺序、建表前置条件、共享 binding 复用规则，以及最小可用运行链路。

## 最新调整
- `lobster-fitting-record-merge`：新增飞书拆分消息归并（bundle）、session 归属判断（append / new_session / pending_confirm）和跨天默认新建规则。
- `lobster-fitting-record-merge` / `lobster-fitting-draft-manager` / `lobster-fitting-bundle-router`：补充仅依赖 skill 的上下文隔离策略，新增 `strict_isolation` / `reset_context` / `force_new_session` / `force_new_bundle` 等约束，默认收紧历史上下文继承，降低串单风险。
- `lobster-fitting-record-merge`：补充强约束——当只有单张图片且无文本时，`bundle_status` 必须为 `pending_media`，`fitting_records` 必须为空，且禁止仅依据图片推断成交。
- `lobster-fitting-record-merge`：新增硬字段 `write_decision`，其中 `PENDING_MEDIA_NOT_READY_TO_WRITE` 作为连接器可直接识别的禁止写入标记。
- `lobster-fitting-bitable-sync`：主表粒度调整为“**一件商品 + 一个客人 + 一个 session = 一条记录**”，主表名调整为 `试衣商品记录`，并补充 `试衣Session索引` 作为可选辅助表。
- `lobster-fitting-bitable-sync`：补充执行约束——`pending_media` 状态下不得写入主表正式记录，并要求连接器识别 `write_decision.allow_write = false`。
- `lobster-fitting-record-merge` / `lobster-fitting-bitable-sync`：补充商品主值优先级约束——本次吊牌 OCR 识别出的商品应优先于历史上下文商品，防止“试穿单品”写成旧商品。
- `lobster-tag-ocr`：收紧职责边界，只输出商品候选信息；单张吊牌图不得直接推断成交、试穿结果或正式入库结论。
- `lobster-fitting-record-merge`：继续强化保守写入策略，默认优先“待补充/待确认”，只有商品主体明确、反馈足够且无冲突时才允许正式写入。
- 新增 `lobster-fitting-bitable-bootstrap`：在写表前检查并补齐 `bitable_binding`、Base、目标表和必要字段，把“可写入”变成正式落表前置条件。
- `lobster-fitting-bitable-bootstrap` / `lobster-fitting-bitable-sync`：明确默认使用项目级共享 binding；已有表则复用，不因不同导购重复建表。
- `lobster-fitting-bitable-sync`：补充主表导购/录入来源字段，新增 `operator_id`、`operator_name`、`source_chat_id`、`source_sender_id`。
- 新增 `lobster-followup-lead-sync`：基于试衣反馈判断是否生成回访线索，并同步更新试衣表与线索回访表，支持输出回访原因、建议触达时机、触达触发条件和建议回访内容；并新增硬约束——不得编造活动、折扣、会员日、到货时间或库存承诺。
- 新增 `lobster-followup-reminder-dispatch`：用于配合统一定时任务扫描线索回访表，派发到期提醒，避免为每条线索单独手工配置定时任务；提醒文案只允许转述已知事实，不得放大经营信息。
- `lobster-followup-lead-sync` / `lobster-followup-reminder-dispatch`：收敛回访提醒为“一次提醒制”，新增 `是否已提醒`、`提醒时间` 字段；`same_day` / `next_day` / `weekend_or_holiday` 必须落成具体提醒时间，`event_triggered` 支持每周固定拉一次待判断清单。
- 新增 `lobster-member-profile-sync`：基于试衣反馈补充会员画像，并同步写入独立的会员画像表，沉淀风格偏好、版型偏好、价格敏感度、未购原因和回访触发偏好等信息。
- `lobster-member-profile-sync`：当识别到会员（如 `is_member=true` 或已拿到 `member_mobile_last4`）时，应默认触发；即使证据较弱，也至少留下最小画像更新痕迹。



## Binding Strategy
- 默认优先使用自动发现机制：先搜索飞书中名称相近且字段匹配的候选表，再由人工确认绑定。
- 确认前只输出候选，不直接写正式业务记录。


## Field Contract
- `references/bitable-field-contract.md`：导购小龙虾相关飞书表字段语义 contract。定义“应有字段”、允许/禁止写入内容、最小可写字段集，以及运行时字段检查规则。


## Link-first Binding
- 当前主方案：人工提供飞书表链接，open claw 首次自动解析 `baseToken / tableId / viewId` 并登记为项目级 binding。
- 后续其他 open claw 直接复用已登记 binding，不再依赖 drive 搜索权限。
- 自动发现/搜索仅作为 fallback。


## Runtime Notes
- 对同一导购 sender，姓名与门店应走 sender profile 继承，不应每条消息重复追问。
- 录音消息默认应走“下载附件 -> 语音转写 -> 结构化整理”链路；只有下载或转写失败时才允许返回失败原因。
