---
id: "f772c2e3-a8c2-4553-9945-ac9dfbd3f5bf"
title: "AgentMesh UI 可用性收口：单一底部输入、可拖分栏、接入成员与服务发现"
description: "本轮 UI 修复的根因（重复分栏与输入实现、测试数据污染生产库）、内置模板与接入成员面板、服务发现边界、真实浏览器验证与可复用经验。"
status: "active"
created_at: "2026-10-29T17:44:35.007Z"
updated_at: "2026-10-29T17:44:35.007Z"
content_hash: "b407e7402af0f1ece9ed378b1a6b21151cd9f80f3dcb42eeab6c71868d409c3c"
source_paths:
  - "ui/app.js"
  - "ui/index.html"
  - "ui/styles.css"
  - "docs/onboarding.md"
  - "test/unit/discovery.test.js"
session_ids:
  - "98262bd0-1652-41bc-b77d-f1ff23dc37c3"
memory_body_ids:
  []
---

# AgentMesh UI 可用性收口：单一底部输入、可拖分栏、接入成员与服务发现

> 来源：`D:\ZISHIKU\AI\AgentMesh` 的 UI 可用性修复会话。相关提交：`ddfd176`（可见可拖分栏）、`558aed0`（内置模板 / 真实子代理树 / 底部派活条）、`c82af48`（接入成员面板与服务发现）、`f7359bb`（单一底部输入框）。本文承接 `66473cbd-52f0-4bcf-a3b1-33fa7e9b339b`（UI 同源接真数据）与 `ec93dea0-9945-4c6a-86a3-2cff7e2b352b`（M1–M10 实现落点），只记录本轮根因、决策边界与验证方式。

## 1. 症状与真实根因

| 用户看到的现象 | 根因（不是“浏览器不支持”） |
|---|---|
| 窗口不能拖大小 / 拖了又弹回 | 新旧两套分栏实现同时存在、互相覆盖；最终只保留一套 |
| 终端区和派活区像两个输入框、焦点混乱 | 三重叠加：旧的 `.term-in` 终端输入行、被移进终端面板的 `.dispatch`、以及额外的 `relocateComposer()` 迁移逻辑 |
| 子代理树里有“没用过却出现”的节点 | 不是前端假数据；是此前测试真实 spawn 出的 `agentmesh-spawn-*` 节点、树、任务、消息遗留在了生产库 |
| 找不到注册应用 / Agent 的入口 | 后端有 spawn / MCP / WS 通道，但前端没有统一接入入口，也没有“本机是否装了某 CLI”的可见性 |

## 2. 本轮落地

- **分栏**：删除重复实现，保留一套；左栏 / 右栏 / 终端高度三根 8px 可见手柄，悬停变色；拖动结果写 `localStorage`，刷新保留；双击手柄复位。实测 1440×900 下左栏 264px → 351px。
- **单一底部输入**：输入框固定在 `.app` 最后（`main` 下方），删除 `relocateComposer()`；`.term-pane .term-in` 隐藏。终端区域改为直接敲键盘（xterm 原生输入），不再额外放输入行。
- **同一输入框三种语义**：普通一句话 → 派活/聊天；`/help`、`/new`、`/status`、`/clear`、`/term` → 系统命令；`!echo hi` → 在当前成员终端执行命令。实测 `!echo CMD_OK` 终端回显 `CMD_OK`。
- **内置模板进 Core**：启动时写入 3 个内置模板（OpenCode / Claude Code / 普通终端），界面从 `GET /api/templates` 读取，不再由前端写死；开终端即自动登记成成员（pane、显示名、online）。
- **接入成员面板**：顶部入口 + 空成员列表 CTA；三块（终端 Agent / 桌面 GUI 应用 / 已有 MCP）；显示 WS 地址、生成一次性接入码、可复制 hello 示例；显示 MCP server 命令 `node src/adapters/mcp-server.js`。
- **服务发现**：新增 `GET /api/discover` 检测 OpenCode / Claude Code 是否安装，未装标“未安装”；接入说明写入 `docs/onboarding.md`，并补 `test/unit/discovery.test.js`。

## 3. 必须说清的产品边界

- AgentMesh **只发现自己打开的终端**；外部已经开着的终端不会凭空出现在成员列表，需在 AgentMesh 内重开，或走 MCP / WS 接入。
- GUI 应用不能“开终端自动发现”，必须由应用主动用一次性接入码连 WS，再发 hello（`display_name`、`node_type: "gui_app"`、`capabilities: ["ui"]`）。
- OpenCode / Claude Code 模板只负责启动命令，本机没装就在终端报错，不代装。
- 拖拽缩放只作用于 AgentMesh 内部面板，不是原生桌面窗口。

## 4. 验证证据

- `GET /api/health` 200；`GET /api/discover` 返回 3 个模板且可用性已检测。
- 真实浏览器（Edge headless + CDP，非 DOM mock）检查：接入面板 5 张卡、3 个终端模板、GUI 地址、token 生成成功、警示文案存在；`app` 子节点为 top / main / dispatch，输入框 parent=app、bottom=800/窗口高 800；`.term-in` display:none；可见输入仅 1 个；控制台无报错。
- 清理测试遗留的 spawn 节点、树、任务、消息后，子代理树显示空态“还没有子代理树”，第一条由用户创建。
- 全量测试 118 pass / 0 fail / 2 skip（真机 e2e 默认跳过）。

## 5. 可复用经验

- 同一区域出现“两套布局 / 两个输入框”，先查重复实现与迁移脚本（`appendChild`、`relocate*`），不要先调 CSS 间距。
- “像假数据”先查库：测试生成的真实节点若未隔离，会被读模型如实渲染；测试应使用隔离库或事后清理。
- 验收 UI 要看真实浏览器的 rect/parent/display 测量和 console 错误，不能只看接口返回或 DOM mock。
- 接入体验的缺口常在“入口 + 可用性 + 限制说明”三件事，而不是协议不存在。
