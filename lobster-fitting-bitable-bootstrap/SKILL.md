---
name: lobster-fitting-bitable-bootstrap
description: 在试衣数据准备写入飞书多维表格前，检查并补齐 bitable_binding、Base、数据表和必要字段。适用于首次部署、binding 缺失、表不存在、字段不完整、或连接器返回 no_binding / missing_binding / invalid_binding 时，用于把试衣台账基础设施稳定建好并返回可写入绑定信息。
---

# 试衣飞书表格初始化与绑定修复

## 目标
在正式写入前，先把飞书多维表格基础设施补齐，并输出稳定可复用的 `bitable_binding`。

## 何时使用
- 首次部署导购小龙虾，尚未创建试衣 Base
- 上游没有 `bitable_binding`
- 已有 `base_token`，但缺少目标表或必要字段
- 查询 / 写入返回 `no_binding`、`missing_binding`、`invalid_binding`
- 需要把“可写入”作为正式写表前置条件

## 输入
- `bitable_binding`（可选）
- `base_name`（可选，默认 `导购小龙虾试衣台账`）
- `folder_token`（可选）
- `time_zone`（可选，默认 `Asia/Shanghai`）
- `schema_version`（可选）
- `expected_tables`（可选）
- `binding_scope`（可选，默认 `project`）

## 默认对象
### Base
- `导购小龙虾试衣台账`

### 必要表
- `试衣商品记录`
- `试衣素材索引`
- `试衣Session索引`

## 工作流
### 1. 先检查 binding
- 默认使用 **项目级共享 binding**：`binding_scope = project`
- 若 `bitable_binding.base_token` 可用，优先在现有 Base 上补结构，不默认新建 Base
- 若系统中已存在可用项目级 binding，则后续所有导购直接复用该 Base
- 不按 `guide_name`、`operator_id`、`sender_id`、`session_id` 创建新的 Base
- 若 binding 缺失或失效，再进入建表流程
- 若输入是 wiki 链接，先解析真实 `base_token`，不要直接把 wiki token 当 Base token

### 2. 再确保 Base 存在
- Base 不存在时，创建新的 Base
- 创建成功后立即记录 `base_token`、`base_url`
- 不为“可能已有但未确认”的情况重复创建多个 Base
- 若已存在可用 Base，但缺表或缺字段，应优先修复现有 Base，而不是重建

### 3. 串行确保三张表存在
- `试衣商品记录`
- `试衣素材索引`
- `试衣Session索引`
- 表存在则复用；表缺失则补建
- 所有 list / create 操作串行执行，避免并发冲突

### 4. 再校验字段并补齐
- 先读取真实表结构，再补缺失字段
- 仅补必要字段，不随意扩展
- 默认字段清单见 `references/default-schema.md`

### 5. 输出稳定 binding
返回结果至少包含：
- `base_token`
- `base_url`
- `tables`（表名 -> table_id）
- `binding_status`
- `schema_status`
- `bootstrap_summary`

### 6. 持久化要求
- 建表成功后，应把 binding 以**项目级单例**形式持久化到系统配置、数据库或状态文件
- 后续写表优先复用已持久化 binding，而不是每次重新建表
- 后续导购进入时，必须先读取这份项目级 binding；读取成功则禁止重复创建 Base

## 核心规则
- 建表与写表分离：本 skill 只负责“确保可写”，不负责写业务记录
- 幂等优先：重复执行时，应补齐缺口，不应制造重复 Base / 重复表
- 默认使用项目级共享 Base：一个项目一套 Base，多导购共用
- 结构优先：先确保 Base、表、字段，再进入 `lobster-fitting-bitable-sync`
- binding 缺失时，不要只返回 `no_binding` 就结束；应主动尝试补齐基础设施
- 若已存在可用 binding，则禁止因新导购触发而再次创建 Base
- 若任一步失败，明确返回失败原因，不伪造 `ready`

## 与其他 skill 的关系
- 上游可来自 `lobster-fitting-record-merge`、`lobster-fitting-draft-manager`、`lobster-fitting-card-fill`
- 建表完成后，再交给 `lobster-fitting-bitable-sync` 生成写表载荷
- 真实建表、建字段、查表结构时，优先配合 `lark-base` 执行

## 输出格式
返回：
```json
{
  "bitable_binding": {
    "base_token": "bascn_xxx",
    "base_url": "https://xxx.feishu.cn/base/bascn_xxx",
    "tables": {
      "试衣商品记录": "tbl_xxx",
      "试衣素材索引": "tbl_xxx",
      "试衣Session索引": "tbl_xxx"
    },
    "binding_status": "ready | partial | missing | invalid",
    "schema_status": "ready | partial | missing_fields | missing_tables",
    "schema_version": "v1",
    "binding_scope": "project"
  },
  "bootstrap_summary": "...",
  "next_action": "go_sync | retry_fix | manual_check"
}
```
