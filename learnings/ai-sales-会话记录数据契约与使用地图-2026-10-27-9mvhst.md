---
title: "AI Sales 会话记录数据契约与使用地图"
author: wuyilin
date: 2026-10-27
tags: [typescript, api, pattern, best-practice]
---

## Context

AI Sales 同时在用户画像和 AI 助手中使用企业微信私聊、群聊及消息历史。接入或扩展会话能力时，应先明确上游数据契约、服务职责和访问顺序，避免把消息标识误作分页边界，或把没有真实来源的摘要字段暴露给页面和模型。

## Solution

### 准确的数据链路

- 成员会话列表：`CustomerProfileService.getSection(..., "member.conversations")` 调用 `CrmChatClient.listSessions`，后者请求 `/public/chat/fetchUserSessionInfos`。返回数据用于发现该成员可访问的私聊和群聊。
- 消息历史：页面 API `POST /api/customer-profiles/[kind]/[id]/conversations/messages` 和 AI 工具最终都进入 `CustomerProfileService.getConversationMessages`，再由 `CrmChatClient.listMessages` 请求 `/public/chat/mixedQueryRecords`。
- 群聊成员：页面 API `GET /api/customer-profiles/[kind]/[id]/conversations/group-members` 和 AI 工具最终都进入 `CustomerProfileService.getConversationGroupMembers`，再由 `CrmChatClient.listGroupMembers` 请求 `/public/chat/getGroupMembersAndDetails`。
- 服务职责：页面和 AI 负责按需发起读取；`CustomerProfileService` 负责加载成员、校验会话或群聊归属；`CrmChatClient` 负责携带 CRM 会话、调用 data-service 并归一化响应。
- `crmv2` 只用于对照真实调用行为和确认上游接口可用，不作为 AI Sales 的数据契约来源，也不复制其页面实现。

### 当前使用地图

- 用户画像：`profile-conversations.tsx` 展示 `member.conversations` 会话列表，按需加载消息历史和群聊成员。
- 页面 API：`sections/[sectionKey]` 提供会话发现，`conversations/messages` 提供历史消息，`conversations/group-members` 提供群成员。
- 服务与客户端：`customer-profile/service.ts` 统一权限和归属校验，`customer-profile/crm-chat.ts` 统一 CRM/data-service 调用与字段归一化。
- AI 会话发现：`get_related_profile_sections` 读取 `member.conversations`，只暴露已验证成员下的会话摘要。
- AI 消息读取：`get_member_conversation_messages` 读取选中会话的一页消息。
- AI 群成员读取：`get_member_conversation_group_members` 仅在身份判断确有需要时读取群成员。

### 推荐访问顺序

1. 先从当前已验证客户关系范围定位成员。
2. 读取 `member.conversations` 发现可访问会话，不接受模型或客户端任意构造的会话 ID。
3. 服务端再次校验成员存在、会话类型和会话归属。
4. 根据问题按需读取消息正文；群成员只在分析参与人身份时读取。
5. 区分真实空数据、未绑定企业微信和上游临时失败，不把失败解释成“没有沟通记录”。

### 分页与证据契约

- `msgId` 是消息唯一标识，用于消息引用和去重；数值数据库 `id` 才是历史分页游标，两者不能互换。
- data-service 的历史条件是 `id < boundary`。即使页面按发送时间展示，下一页游标也应取当前页全部合法数据库 `id` 的最小值。
- 分页游标严格使用 `1..2147483647` 范围内的整数 number；字符串、复合 `msgId`、小数和越界值应被拒绝，不转换或回退。
- AI 消息读取在数据源请求阶段显式限制为每页 20 条，同一请求最多读取三页。文本正文是分析证据，应按实际剩余预算保留；非文本消息只保留类型、时间、发送者和已有可见说明，不推测媒体或附件内容。
- 会话摘要只消费上游真实提供的字段。当前上游没有 `sessionStartAt`、`sessionEndAt`、`latestMessageAt`、`messageCount`，因此不读取、不推算，也不返回 `null` 占位。

## Lessons

- 会话接入的第一原则是先发现并校验归属，再读取内容；权限边界应集中在服务层复用。
- 消息标识、分页主键、发送时间分别承担不同职责，字段名相似不能替代接口和查询语义。
- AI 会话读取应在源头控制页大小，并把文本正文作为优先证据；无法保留最低正文时应显式返回预算不足。
- 新增摘要字段前必须确认上游 DTO 和实现确实提供该字段，没有来源的字段不应进入共享类型、页面响应或 AI 数据集。

## Related Skills

- technical-design
- backend-coding-standards
- self-test-design
