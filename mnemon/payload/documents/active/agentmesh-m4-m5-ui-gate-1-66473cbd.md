---
id: "66473cbd-52f0-4bcf-a3b1-33fa7e9b339b"
title: "AgentMesh M4/M5 落地与 UI 同源接真数据（Gate-1 未关）"
description: "M4 真机 Herdr e2e、M5 派活/看板/撤回后端、Task 5.3 UI 同源接真数据的根因与解法、语义约束、验证现状及 Gate-1 未收口项。"
status: "active"
created_at: "2026-10-29T02:35:04.778Z"
updated_at: "2026-10-29T02:35:04.778Z"
content_hash: "bf4cbebf043d68f4ab4699e38c485c25a35ec8785eea262a5f43440f74779316"
source_paths:
  - "src/core/tasks.js"
  - "src/core/delivery.js"
  - "src/api/http.js"
  - "src/api/ws-hub.js"
  - "ui/app.js"
  - "test/e2e/m4-spawn.e2e.test.js"
  - "test/unit/m5.test.js"
  - "docs/evidence/gate-1.md"
session_ids:
  - "88ee8606-c3ca-4a2a-a3e0-729634b1a36f"
memory_body_ids:
  []
---

# AgentMesh M4/M5 落地与 UI 同源接真数据（Gate-1 未关）

> 来源：工作区 `D:\ZISHIKU\AI\AgentMesh`。相关提交：`57ac952`（M4 真机 e2e）、`7bcc00b`（M5 派活/看板/撤回）、`d23511d`（UI 接真数据与同源托管）。关联文档：`938fa791-fc48-45e8-9a87-8a67b2a6ab92`（M1–M10 计划审核）、`74ed2e47-2a1d-4598-a22a-a03090a8d351`（PRD 4.1 交接基线）。

## 1. 本轮已落地

- **M4 真机**：用 `parent_user` spawn → pane 存在 → 子节点/树落库 → destroy 关闭 pane、节点标记 destroyed，替代原先的 fake Herdr；e2e 见 `test/e2e/m4-spawn.e2e.test.js`，`AM_E2E=1 npm run test:m4` 为 5 pass。
- **M5 后端**：新增 `src/core/tasks.js`。成员 send/request 可派活；组派只发不等（`group + wait=true` 返回 422）；任务、起点消息、trace 在同一条路径落库；看板读模型含 owner、最后消息时间、当前步骤、失败原因；任务支持 close / reopen / handoff；撤回窗口 10 秒。
- **Task 5.3 UI**：删除 mock 分支，派活走 `POST /api/tasks`，记录读 `GET /api/messages`，撤回走 `POST /api/messages/{id}/revoke`，看板读 `GET /api/tasks?board=1`，"沿用上次设置"默认生效。

## 2. UI 打不开真数据的根因与解法（关键）

- 根因不在 `ui/app.js` 的业务逻辑，而在 `file://` 打开导致的跨源请求失败，且界面没有本机身份。
- 解法：由 Core 同源托管 `/`、`/app.js`、`/styles.css`；UI 点"连接本机"时自举 token 并注册 `ui-local`（gui_app）节点，该本机身份从成员列表隐藏。
- 结果：不需要单独起 UI 服务器、不需要 CORS；浏览器直接访问 `http://127.0.0.1:7600/` 即可操作真实 Core。真实浏览器（Edge headless + CDP）验证，而非 DOM mock。
- 过程中踩到的实现坑：`api()` 内重复声明 `const body` 导致 `SyntaxError`，已修正。

## 3. 必须保持的语义约束

- 组播只能单向发送（FR-4.6），不允许 group + wait。
- 撤回仅限 10 秒内且消息仍为 pending；已 delivered 返回 409，非发送者返回 403。
- handoff 摘要上限 200 字（M6 继续收紧），超长应拒绝并提示。
- 实时更新禁止轮询（FR-6.7），应走 WS 推送；任务详情需能回看派活记录（FR-10.5）。

## 4. 验证现状与证据

- 全量 `npm test`：79 pass / 0 fail / 2 skip（两条真机 e2e 默认跳过）；M5 单测 4/4。
- 浏览器实测：连接本机 → 派给 demo-a → "UI 派活测试"；SQLite 中出现 task（running，owner=demo-a）与关联的 send 消息（ui-local → demo-a，task_id 已关联）；看板显示 1 张进行中卡片；刷新后派活目标仍为 demo-a。
- 证据文件：`docs/evidence/gate-1.md`、截图 `docs/evidence/gate-1-ui.png`。

## 5. 未收口

- `ws-hub` 实时推送未接，当前是操作后 fetch 刷新，不满足"2 秒内自动出现"；三张录屏未补，因此 **Gate-1 未关闭**。
- 下一阶段：M6.1 看板读模型/交接摘要/双向跳转，M6.2 看板 UI 接真 + 2 秒推送，再走 Gate-2。
- 判断边界：API 与单测通过不等于验收门通过；Gate 需真实 UI 路径与证据。

## 6. 可复用经验

- 前端"读不到真数据"先查协议与身份（跨源、token、节点注册），再查业务代码。
- 验收证据要同时包含真实浏览器操作、SQLite 行变化、界面结果；缺一项就明确标"未收口"。
