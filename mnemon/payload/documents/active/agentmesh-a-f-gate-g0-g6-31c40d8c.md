---
id: "31c40d8c-76a0-41e4-a1bb-7e710c61711e"
title: "AgentMesh 商业可用性验收结论与硬化计划（A–F / Gate G0–G6、零模型控制面门禁）"
description: "商业可用性实测判定、五项硬缺口（真实 Agent 握手、终端恢复、投递可靠性、权限审计、长稳压测）、Gate 顺序与借鉴产品、计划审核修正（hello_ok、agentmesh_* 工具名、Windows 进程终止）、零模型控制面防回归测试与触发器迁移 15。"
status: "active"
created_at: "2026-10-29T18:29:05.565Z"
updated_at: "2026-10-29T18:29:05.565Z"
content_hash: "5c71d46dab88219ed57e2ecef0cb016bdca6185c3c84e045d9cc07f56232374f"
source_paths:
  - "docs/superpowers/plans/2026-10-30-commercial-hardening.md"
  - "docs/token-free.md"
  - "test/unit/token-free.test.js"
  - "docs/superpowers/plans/2026-10-29-agentmesh-m1-m10.md"
  - "src/api/ws-adapter.js"
  - "src/adapters/mcp-server.js"
  - "src/core/db.js"
session_ids:
  - "1d0cf0a3-30a5-4948-b1db-9ed0144fc337"
memory_body_ids:
  []
---

# AgentMesh 商业可用性验收与硬化计划（工作流 A–F、Gate G0–G6）

> 来源：工作区 `D:\ZISHIKU\AI\AgentMesh`。基线为 M1–M10 已跑通本地主链后的商业门槛验收，产物 `docs/superpowers/plans/2026-10-30-commercial-hardening.md`（484 行）。相关文档：`938fa791-fc48-45e8-9a87-8a67b2a6ab92`（M1–M10 计划审核与项目骨架）。本文只记结论、判据与理由，不复述计划全文。

## 1. 实测主链（已验证事实）

真实终端跑通一条端到端链路：创建终端 → 发消息 → 写入终端执行 → 派活 → 写入终端执行 → 任务转 running。证据：`{"spawn":true,"message_executed":1,"task_status":"running","task_executed":1,"sawMsg":true,"sawTask":true,"msgCount":1}`，health=200，全量测试 121 pass / 0 fail / 2 skip（123 用例）。

结论：证明的是**本地 shell 主链可跑**，不等于商业可用——测试只覆盖已实现代码。

## 2. 商业判定（2026-10-30）

| 能力 | 判定 |
|---|---|
| 本地单用户工作台、shell / 自定义命令 Agent | 原型可用（后者限受控命令） |
| OpenCode / Claude Code 真实接入 | 只启动命令，未验证模型级 A/B 对话，未闭环 |
| MCP / WS 通道 | 有协议与测试，需 Agent 自己配置与握手 |
| 终端重启恢复 | 进程内 pane，重启即失效，不达标 |
| 消息重连 / 幂等 / 长稳 | 有基础 ack/queue，无断线重放与压测，不达标 |
| 多用户 / 权限 / 审计 | 无，团队场景不达标 |
| 可观测性 / 负载 / 故障 | 仅 health/diagnostics/GC；SQLite 单写者，无压测 |

判断：**个人或小团队本机演示、受控自动化可用；对外商业交付不可用。** 最危险的缺口不是 UI，而是"看起来能连 Agent，实际依赖人肉配 MCP / 终端注入"。

## 3. 硬化工作流与 Gate 顺序

G0 基线 → G1 Agent 接入 → G2 终端恢复 → G3 投递可靠性 → G4 权限审计 → G5 长稳压测 → G6 商业功能签收；F（GUI 原生嵌入 / 安装升级备份）P2 后置，不阻塞 G1–G5，但 G6 前要有排期。

- **A 真实 Agent 接入**：发现本机配置 → 用户确认 → 自动写 MCP 配置 → 启动 → 握手（MCP `initialize` → `tools/list`）→ 结构化收发 → 分步降级并给出可见原因。
- **B 终端会话恢复**：常驻 terminald 托管 PTY；Core 重启 reattach 并保留滚动缓冲；terminald 崩溃时 pane 标 orphan 可重建；孤儿回收与进程树终止。
- **C 投递可靠性**：至少一次 + 按 `msg_id` 幂等、退避重试、可见性超时、死信、游标重放、取消、背压、队列上限。
- **D 权限与审计**：工作区 / 角色 / token 作用域、谁能给谁发、自定义通信线是否强制路由（`link_enforced`）、审计留痕用哈希链。
- **E 长稳**：故障矩阵（断网、Agent 崩溃、终端卡死、队列打满）、10k 消息压测、24h soak、SLO 与 `/api/metrics`。
- **F 后置**：Tauri/Electron 式原生宿主（不用 iframe 假装嵌入）、安装/升级/备份与回滚演练。

每个 Gate 产出证据文件 `docs/evidence/commercial-gN.md`；控制面协议变更才重跑真实冒烟，其余用假适配器。

## 4. 借鉴的成熟产品做法

MCP 接入看 Claude Code / OpenCode / Cursor；终端会话看 tmux / zellij 的 client-server 与 reattach；投递看 NATS JetStream / RabbitMQ 的 ack、可见性超时、死信、重放；权限审计看 GitHub / Linear 的工作区与审计；长稳看 Google SRE / Chaos Mesh / Toxiproxy / k6；GUI 用 Tauri / Electron 原生宿主。

## 5. 计划审核修正（对照现有代码与协议）

| 原表述 | 修正 |
|---|---|
| WS 握手 `hello -> welcome` | 实际为 `hello -> hello_ok`（`src/api/ws-adapter.js`） |
| MCP 工具名为裸 send/request | 实际带前缀：`agentmesh_send / recv / ack / status / request / wait_reply / reply`，`cancel` 为目标项 |
| Windows 用 SIGTERM / kill -9 | Windows 用 Ctrl+C / TerminateProcess / Job Object；POSIX 才用 SIGTERM / kill -9 |
| "Core 不执行命令" | Core 不直接执行，进程由 terminald 托管 |
| token 含义混用 | 区分**模型 token**（费用）与**接入 token**（鉴权） |
| GUI 三平台承诺过满 | Windows 原生嵌入待校验，macOS 多为外部窗口捕获，Wayland 受限，先做平台 spike |
| 真实冒烟预算过死 | 每 Gate 每个 Agent 一次真实冒烟，失败重跑须记录预算 |

另：旧 M1–M10 计划曾被误截断，已恢复 478 行完整版并加"当前已改为内置 PTY、商业计划另见"的状态注记。

## 6. 用户硬约束：代码能做的不要消耗用户 token

原则落地为**零模型控制面**，并做成防回归门禁而非口号：

- 路由、投递、握手、权限、审计、通信流、任务看板、健康检查、GC 全部本地 SQLite/WS 代码实现，不调用 LLM。
- 消息原样传递，不总结、不改写 payload；接入模板与诊断报告用确定性代码生成。
- 测试默认假适配器；真实 Agent 冒烟受 Gate 预算约束；可选 AI 功能必须显式开启、默认关闭。
- 门禁 `test/unit/token-free.test.js` 扫描 `src/`：模型 SDK 依赖（openai、@anthropic-ai/sdk、@google/generative-ai、langchain）、模型 API 域名、`chat/completions` / `messages.create` / `generateContent` 调用，以及 `package.json` 新增模型 SDK——违反即全量测试失败。说明文档 `docs/token-free.md`。
- 边界须明说：AgentMesh 自身零模型调用；用户选 Claude Code / OpenCode 时其执行任务产生的厂商 token 属 Agent 成本；要完全零模型 token 用普通终端 / 本地脚本模板。

## 7. 顺手修掉的硬问题

`messages_no_update` 触发器把 `revoked_at` 也拦住，撤回报 `messages is append-only`。迁移 15 重建触发器，仅允许撤回标记列变更，正文/路由等列仍不可改，保留"消息只追加、撤回留痕不删消息"的语义。

## 8. 提交

`2d87c55`（通信流操作：继续/暂停/继续服务/撤回/重试）、`35f1751`（撤回后隐藏重试）、`53e97c2`（硬化计划）、`494cb41`（恢复完整 M1–M10 计划）、`34460cd`（对照协议审核计划）、`cc50300`（零模型控制面门禁 + `docs/token-free.md`）。
