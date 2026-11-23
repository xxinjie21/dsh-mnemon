---
id: "e6239e87-2740-4fa7-87cc-dc41d9ad1950"
title: "AgentMesh 零 token 通信流观测与流内操作（暂停/撤回/重试）"
description: "零 token 本地通信流的数据来源与不产生 token 的机制、流内五类操作的真实语义、撤回/重试服务端约束，以及 schema 15 修复 messages 追加写触发器误拦 revoked_at 的根因与验证。"
status: "active"
created_at: "2026-10-29T18:13:51.729Z"
updated_at: "2026-10-29T18:13:51.729Z"
content_hash: "59fcd59418fa3808442bac023b62e81ffa00c8a8884600c87deaf92f74423a7e"
source_paths:
  - "src/core/delivery.js"
  - "src/core/db.js"
  - "src/api/http.js"
  - "ui/app.js"
  - "ui/index.html"
  - "ui/styles.css"
  - "docs/flow.md"
session_ids:
  - "0d81d3d1-2ff8-4119-9eb3-b6d1bea7f5ad"
memory_body_ids:
  []
---

# AgentMesh 零 token 通信流观测与流内操作（暂停/撤回/重试）

> 工作区 `D:\ZISHIKU\AI\AgentMesh`。本轮提交：`de3bd26 feat(ui): local communication flow without model tokens`、`2d87c55 feat(ui): flow actions continue pause recall retry`、`35f1751 fix(ui): hide retry for revoked flow message`。
> 关联既有文档：`dd5cce32-7258-498e-8f52-ac5a27a867e3`（用户自定义通信线 links 的边界裁决与落点）、`66473cbd-52f0-4bcf-a3b1-33fa7e9b339b`（M4/M5 落地与 UI 接真数据）、`d373d32b-0b73-4671-8066-ccf944a194b2`（一期设计定稿：三路由、append-only 会话/trace 取向）。

## 1. 需求与定位

用户提出两点：

1. 各应用程序、agent 之间要**可视化交流过程**，但该过程**不要使用 token**；
2. 在通信流中要能对节点做市面上常见 Agent 产品具备的操作：**追加对话、暂停、撤回**等。

定位：这是**展示层与本地可观测性**改动，不是新的编排层。数据全部来自本地 SQLite 与 WebSocket 事件，不调用模型、不生成自然语言摘要，因此"看过程"本身不产生 token 费用。

## 2. 零 token 的数据来源

| 数据源 | 提供的信息 |
|---|---|
| SQLite `messages` / `traces` | 谁发给谁、发送/提问/回复模式、正文、时间、trace 批次 |
| WebSocket `message_executed` / `message_revoked` | 消息是否真正写入目标终端、是否被撤回 |
| 本地计算渲染 | 时间格式化、批次分组、状态文案，直接渲染，不做模型总结 |

之所以零 token：没有摘要模型、没有自然语言生成、没有第二轮 LLM 调用。相反，如果让模型"复盘"过程，既慢又产生费用。

## 3. 明确边界（不可放大）

- 历史消息若未保存执行事件，只能显示"已发出"，**不能倒推**"已执行"；
- 通信流是本地可观测性视图，不是编排层；
- 它不是模型生成的"复盘"，所以没有 AI 解释，但也没有 token 账单；
- 界面顶部说明固定为"本地消息与执行事件 · 不调用模型，不消耗 token"。

## 4. 流内操作的实际语义

| 操作 | 真实行为 |
|---|---|
| 继续对话 | 把底部输入框的发送目标切到该成员，直接发下一条 |
| 暂停服务 | 向目标终端发 Ctrl+C 打断当前任务，并把后续消息改为**排队**（`deliveryMode=queue`） |
| 继续服务 | 恢复自动投递，并冲刷该成员的排队消息；按钮由"暂停服务"切换而来 |
| 撤回 | 本地界面以操作者身份**强制撤回未确认**消息 |
| 重试 | 以**原发送者身份**重发，生成新的消息与投递记录（不改历史） |

边界：

- 「暂停」不是杀进程：只打断当前活并让新消息排队；真正关闭成员仍走关闭操作；
- 「撤回」只能撤回未确认消息，对方已确认的仍撤不回；
- 「重试」生成新消息，历史不可改写；已撤回消息不显示重试按钮（`35f1751`）。

## 5. 服务端语义约束（`src/core/delivery.js`）

撤回 `revokeMessage(db, msgId, nodeId, { force })`：

- 404 `message_not_found`；403 `forbidden`（只能撤回自己发的）；409 `nothing_to_revoke`（无投递记录）；409 `already_acked`（对方已确认处理，撤不回）；
- 默认窗口 10 秒，超窗且已 delivered 返回 409 `revoke_window_expired`；
- `force` 强制撤回仅作用于 `pending`/`delivered` 的投递，并写 `messages.revoked_at` 留痕，不删消息。

重试 `retryMessage(db, msgId, nodeId)`：404/403 校验同撤回；已撤回（`revoked_at`）返回 409 `message_revoked`；重发时带 `ref_msg_id` 与 `retry_of_msg_id` 指向原消息，保持重试谱系。

## 6. 关键缺陷与修正：append-only 触发器错误拦截撤回

现象：点击撤回返回 `messages is append-only`，撤回失败。

根因：schema 14 的 `messages_no_update` 触发器把 `revoked_at` 也纳入了"禁止 UPDATE"的列范围，而撤回的实现正是写 `revoked_at`。这与既有设计原则冲突——追加写是硬约束（正文/路由/时间戳不可改写），但撤回必须能留痕。

修正（迁移 version 15）：重建 `messages_no_update`，仅在 `msg_id`、`payload`、`from_node_id`、`to_node_id`、`timestamp`、`trace_id`、`mode` 变化时 `RAISE(ABORT, 'messages is append-only')`，**放行 `revoked_at`**。数据库注释同步说明"唯一例外是撤回标记，撤回要留痕，不删消息"。

教训：append-only 表上的状态标记字段必须在 schema 设计时显式列入"允许更新"白名单，否则业务写入会被自身的不可变约束挡住；迁移需同时更新触发器与注释，并用真实 UI 路径验证，而不是只看单测。

## 7. 实现落点与验证

- UI：`ui/app.js`（通信流渲染、按 trace/模式的筛选、流内操作按钮、WS `onmessage` 触发同步）、`ui/index.html`（右侧 tab「通信流」）、`ui/styles.css`；接口与静态资源在 `src/api/http.js`（静态 UI 响应带 `Cache-Control: no-store`，避免旧 CSS 缓存）。
- 文档：`docs/flow.md`（随 `de3bd26` 新增）。
- 实测（真实浏览器 + CDP）：流内出现"继续对话 / 暂停服务 / 撤回 / 重试"；点击暂停后按钮变"继续服务"、节点 `deliveryMode=queue`；点击撤回后 toast「消息已撤回」、状态变"已撤回"、撤回按钮消失；控制台无报错。
- 全量测试：119 pass / 0 fail / 2 skip。
- 服务：`http://127.0.0.1:7600`，schema_version=15（撤回修复后重启验证）。

## 8. 关键路径

`src/core/delivery.js`、`src/core/db.js`（迁移 15 与 `messages_no_update` 触发器）、`src/api/http.js`、`ui/app.js`、`ui/index.html`、`ui/styles.css`、`docs/flow.md`。
