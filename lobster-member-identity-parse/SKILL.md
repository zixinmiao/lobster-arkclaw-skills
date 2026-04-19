---
name: lobster-member-identity-parse
description: 识别试衣场景中的会员身份信息。适用于导购文字、语音转写、上下文中出现会员、手机号尾号、会员备注等线索时，判断是否为会员试衣，并提取会员尾号后四位与状态字段。
---

# 会员身份识别

识别是否为会员，并输出结构化结果。

## 输出目标
输出 `result`：
- `is_member`: `true` | `false` | `null`
- `member_mobile_last4`: 字符串
- `member_identity_status`: `confirmed` | `mentioned_without_last4` | `not_member` | `unknown`
- `member_notes`: 字符串
- `confidence`: 0 到 1

## 判定规则
- 明确提到“会员”且给出尾号后四位：`confirmed`
- 提到“会员”但没有尾号：`mentioned_without_last4`
- 明确表示不是会员：`not_member`
- 信息不足：`unknown`
- 只提到数字但无足够上下文，不要直接认定是会员尾号。

## 输出格式
返回：
```json
{"result": {...}}
```
