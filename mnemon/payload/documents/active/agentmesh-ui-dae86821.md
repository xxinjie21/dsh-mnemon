---
id: "dae86821-277a-4e0f-9fb5-e38725023ac9"
title: "AgentMesh 会话成员持久化落地与真实数据优先 UI 约定"
description: "迁移 16 task_members 与会话成员接口、原型会话链路切真数据、完整接入面板恢复、rail/交接栏空态约定、提交链与教训。"
status: "active"
created_at: "2026-10-30T05:43:14.155Z"
updated_at: "2026-10-30T05:43:14.155Z"
content_hash: "92be4b0b7961316c1966f8e5f3760ce25bfe48d2da0c3f99500b0d755ff5a08d"
source_paths:
  - "src/core/task-members.js"
  - "test/unit/task-members.test.js"
  - "prototype/prototype-bridge.js"
  - "prototype/AgentMesh-原型.html"
  - "docs/frontend/agentmesh-frontend-contract.md"
session_ids:
  - "6f566dfa-114e-47d7-bfb8-4092cf65bcb9"
memory_body_ids:
  []
---

# AgentMesh 会话成员持久化落地与「真实数据优先」UI 约定

> 工作区 `D:\ZISHIKU\AI\AgentMesh`。2026-10-30 同一会话的后继阶段：补齐会话成员持久化、把原型会话链路从本地假状态切到真实后端、恢复完整接入面板，并纠正「隐藏入口」的处理方式。
> 前置与相关文档：会话优先 IA 与「成员持久化缺口」见 `8f0c84e1-decb-431e-8be4-a0d14d9910ad`；桥接映射与路由/异常契约见 `657ebab7-c6e8-46eb-a5e8-72c38c2890a0`；范围收敛与「改原型本体、不做副本」见 `672f0d19-ab57-44fc-892f-4224d5dfb5b9`。本文只记录本轮新增的持久化模型、接口、前端接线与 UI 约定。

## 1. 用户约束（本轮明确）

- 会话成员的「原本的前端入口」必须保留完整形态：终端 Agent 三入口（OpenCode / Claude Code / 普通终端）、桌面应用 GUI（WS 地址 + 接入码 + hello 示例）、已有 MCP 的 CLI 说明、以及「AgentMesh 只发现自己打开的终端」的提示。
- 打开终端后应自动出现在成员列表，不需要手工登记。
- **不得靠隐藏区块来消除假数据**：「该有显示的要显示，显示为空或者友好提示」——壳保留、内容按真实数据渲染，无数据时给空态文案。

## 2. 后端：迁移 16 与会话成员接口

- 新增迁移 16，建表 `task_members(task_id, node_id, role default 'member', created_at)`，主键 `(task_id, node_id)`，外键级联到 `tasks` 与 `nodes`（`ON DELETE CASCADE`），索引 `idx_task_members_node`（按 node 反查）。
- 随之 `LATEST_SCHEMA_VERSION = 16`；服务启动日志实测 `schema_version=16`、库文件 `data/agentmesh.db`、`内置模板已就绪`。
- 新增接口：`GET /api/tasks/:id/members`、`POST /api/tasks/:id/members`、`DELETE /api/tasks/:id/members/:node_id`。
- 成员返回体为「任务 owner ∪ `task_members` 行」，每项含 `node_id / role / created_at / display_name / node_type / status`。
- 领域逻辑独立成 `src/core/task-members.js`（不与 HTTP 层耦合），配套 `test/unit/task-members.test.js`。

## 3. 前端：会话链路全部走后端

- 左侧「会话」列表 = 真实任务列表；会话头参与者 = `task_members` + 任务 owner；聊天/发送 = 真实任务消息；暂停/继续 = 节点暂停/恢复。
- 「+ 新建会话」：创建真实任务（`POST /api/tasks`），再对其余所选成员逐个 `POST /api/tasks/:id/members`。
- 「+ 成员」：从已接入节点中选择，写 `task_members` 持久化。
- 「接入成员」：请求 `/api/spawn`（`parent_user: true`、`template_id`、`scope: 'children'`）真实 spawn，子节点自动注册进成员列表。
- 移除了桥接层里写死的 14 项 `CATALOG` 假成员；无成员时显示空态与「接入成员」，不再以假模型充数。
- 静态壳同样处理：左侧「模块 / AI 助手 / 会话列表」与画布底部交接栏恢复显示，仅按真实数据渲染或给空态；画布底部「示例事件」字样改为「材料交接」。

## 4. 接入成员面板（恢复完整形态）

- 面板数据来自真实接口：模板与可用性取 `/api/discover`（或 `/api/templates`），GUI 侧显示真实地址 `ws://127.0.0.1:7600/adapter/ws`，接入码由后端生成（实测形如 `am_Oxhph…`，仅显示一次），MCP 命令为真实 `node src/adapters/mcp-server.js`。
- 曾出现回退：某次简化把面板退化成「模板列表」，丢失 GUI 接入码、MCP 说明与「只发现自己打开的终端」提示；本轮按原内容恢复（提交 `8743aac`）。
- 关键语义：控制面全部本地执行、不调用模型；只有用户选中的 Agent 自己可能产生模型 token。

## 5. 验证证据

- 全量测试：`125 pass / 0 fail / 2 skip`；成员模块单测 `1 pass`。
- 服务启动：`api listening: http://127.0.0.1:7600`、`schema_version=16`、终端子系统 `pane` 残留 0 个已标离线；`/api/tasks/:id/members` 实测返回 2 条成员，新建会话/加成员均 200。
- 浏览器实测（无头）：新建会话后会话数 +1、成员 chip 与后端一致；面板内「终端里的 Agent / 桌面应用（GUI）/ MCP 的 CLI / 外部终端提示」四项齐全，token 生成成功；左侧 rail 与交接栏为 `display:flex` 且显示空态文案；控制台报错 0。
- 结束时清空测试数据，库中仅保留本机身份。

## 6. 提交链

- `8f862b1` feat(frontend): persist sessions and member management
- `dfdddff` feat(frontend): persist session members and bind session UI
- `bfa7e7a` feat(frontend): add real member onboarding from templates
- `bcf7849` fix(frontend): hide static module/assistant/session rails（做法后被用户否决）
- `8743aac` feat(frontend): restore full onboarding panel
- `ff359b0` fix(frontend): show real rail and friendly empty states

## 7. 仍在缺口内的入口（不得假装可用）

通知、系统状态/诊断、通信线管理与通信流独立视图、嵌入式终端、消息撤回/重试；「模块」尚无后端实体，只能给友好空态。`/classic` 为归档旧界面，不再新增功能。

## 8. 教训

- 去假数据的正确做法是「换真实数据源 + 空态」，而不是删掉入口；隐藏整块壳会被用户当作功能缺失退回。
- 原型既有文案本身就是需求说明（接入面板文案、GUI 接入码说明、MCP 提示），简化时逐条比对，不要以「能跑」为由砍内容。
- 前端选择态与后端 ID 的映射要显式验证（key→node_id 映射曾导致选第 2 个成员却加错节点），实测应以接口返回的成员列表为准，而不是界面上看 chip 数量。
- 验证要用真实浏览器取 DOM/接口证据（前端实测脚本 + 截图），不要以单测通过代替界面行为确认。
