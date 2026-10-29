---
id: "938fa791-fc48-45e8-9a87-8a67b2a6ab92"
title: "AgentMesh M1–M10 计划审核结论与项目骨架（C16、八处修正）"
description: "AgentMesh 执行计划的 8 处执行偏差修正及理由（PRD 路径、:memory: 不支持 WAL、M1 不占端口、paneSplit 开 pane、group+wait 422、yazl 依赖、grep 假阳性）、C16 依赖约束、已落地的目录职责边界与四个有意不做项、下一步 Task 1.1。"
status: "active"
created_at: "2026-10-28T16:44:03.094Z"
updated_at: "2026-10-28T16:44:03.094Z"
content_hash: "059ea49c813b67cb6d4b9abea113e50d82643f0d8c4affe47b0cc7c8a2771b54"
source_paths:
  - "docs/superpowers/plans/2026-10-29-agentmesh-m1-m10.md"
  - "docs/AgentMesh.md"
  - "package.json"
  - "README.md"
  - "docs/evidence/README.md"
session_ids:
  - "dd98ca43-44c9-4341-9694-b5dca40fa0ff"
memory_body_ids:
  []
---

# AgentMesh：M1–M10 执行计划审核与项目骨架

> 来源：工作区 `D:\ZISHIKU\AI\AgentMesh`。审核对象 `docs/superpowers/plans/2026-10-29-agentmesh-m1-m10.md`（472 行 / 29,980 bytes），需求基准 `docs/AgentMesh.md`（PRD v4.1，766 行 / 53,220 bytes）。相关文档：`cdd9b45a-3b0f-4950-9924-71bb5628b1f4`（架构原则与三阶段十里程碑）、`74ed2e47-2a1d-4598-a22a-a03090a8d351`（PRD 4.1 交接要点）。本文只记审核结论、落地结构与执行约束，不复制计划全文。

## 1. 审核定位

计划不是"做什么"的清单，而是下一位 Agent 可照路径执行的脚本。审核标准因此是"照着做会不会第一步就报错"，不是文笔。共修 8 处会导致执行偏差的问题，并新增一条全局依赖约束 C16。

## 2. 八处修正与理由

| # | 原问题 | 修正 | 理由 |
|---|---|---|---|
| 1 | Spec 写 `AgentMesh.md`（根目录） | 改为 `docs/AgentMesh.md` | PRD 实际位于 `docs/`，路径错则第一步读不到需求 |
| 2 | 文件树漏 `docs/evidence`、`examples/gui-reference`、`README`、`.gitignore`、`test/herdr-mock.js` | 按实际目录补齐 | 计划里的证据路径与 mock 位置必须真实存在，否则 Gate 证据无处落盘 |
| 3 | Task 1.1 用 `:memory:` 断言 WAL | 改用 `mkdtempSync` 临时文件库并 `db.close()` | SQLite 内存库不支持 WAL，该测试必然假失败 |
| 4 | Task 1.1 Step 5 要求输出 `core listening 127.0.0.1:7400` | 改为 `db ready: <path> · schema_version=1`，本步不占端口 | HTTP 监听属 M2 Task 2.1；提前占端口会让 M1 越界且验收口径混乱 |
| 5 | Task 1.3 e2e 用 `agentPrompt` "打开 pane" | 改为 `paneSplit` 开 pane → `agentPrompt` 发提示 → `paneRead` | Herdr 中开 pane 是 `paneSplit`，`agentPrompt` 只负责发提示，混用会卡住 e2e |
| 6 | Task 5.1 允许 `group + wait=true` | 组派只能 send，`wait=true` 返回 422，并补测试 | 违反 PRD FR-4.6（组播只能单向发送） |
| 7 | Task 10.1 要生成诊断 zip 但技术栈无 zip 依赖 | 技术栈加 `yazl`，Task 10.1 明确写入 package.json | 依赖缺失会让 M10 临场改技术栈 |
| 8 | Task 10.3 用 grep 源码查术语 | 改为 CDP / 无头浏览器读可见文本，源码字段名允许英文 | grep 会误命中 `traceFilter` 等字段名，产生假阳性 |

新增 **C16**：任何新增依赖必须写入 `package.json` 并锁版本，不全局安装。防止执行者用全局包造成"本机能跑、换机即崩"。

## 3. 已落地的项目骨架（一处一职责）

```text
AgentMesh/
  package.json     # 仅脚本：start / db:check / test / test:unit / test:e2e
  README.md        # 入口：先读 PRD 与计划
  .gitignore       # node_modules / data 库 / 本地日志
  docs/
    AgentMesh.md                 # PRD v4.1，唯一需求文档
    superpowers/plans/           # M1-M10 执行计划
    evidence/                    # Gate-0~4 证据（gate-N.md）
  ui/              # 现有静态原型（全 mock，零构建）
  data/            # SQLite 文件（git 忽略）
  src/core|herdr|api|adapters|bin/
  test/herdr-mock.js、unit/、e2e/
  examples/gui-reference/        # 最小 WS 样板应用
```

模块职责边界（写进计划的文件结构章节，防止职责漂移）：`core/db.js` 唯一开库处（WAL + busy_timeout 5000 + foreign_keys ON + 单调迁移）；`core/bus.js` 路由校验/seq/trace/追加写；`core/delivery.js` 每节点队列/补投/ack/幂等/撤回；`herdr/client.js` 只走 socket API；`api/*` 与 `adapters/mcp-server.js` 只翻译消息，不承载执行。

## 4. 有意不做的四件事

- **未 `git init`**：计划 Task 1.1 含此步，但引入版本管理应由执行者或用户确认，不在审核阶段擅改仓库状态。
- **未写实现代码**：`src/` 仅目录占位，`db.js`、`bin/agentmesh.js` 留给 Task 1.1，避免"骨架即实现"的假进度。
- **未装依赖**：`better-sqlite3` 留 M1，`ws`/MCP SDK 留 M2，`yazl` 留 M10，均按 C16。
- **未把审核过程写进 PRD**：PRD 保持描述性，工作记录只留在对话与本文档。

## 5. 执行顺序与验收口径

顺序固定 M1 → M2 → M3（Gate-0），再 M4 → M5 → M6 → M7，M8–M10 其后；不过门不进下一阶段。每个里程碑的"完成"必须有命令输出、截图或真实交互证据，禁止把 mock 说成已打通。下一步起点是 Task 1.1（脚手架 + SQLite 迁移骨架），验收输出为 `db ready: <path> · schema_version=1`。

## 6. 五个最容易咬人的测试点（计划 Review Focus）

1. 撤回边界：已 delivered 调 revoke 必须 409，绝不能把 delivered 改成 revoked（M5 Task 5.2）。
2. 组播不补投、单播必补投且按 seq 顺序（M3/M4）。
3. 同一 `msg_id` 补投两次只处理一次、界面只显示一条（M3 Task 3.3）。
4. `to_node_id / group_id / spawn_tree_id` 多填或全空必须 400；request 指向 group/tree 必须 422（M3 Task 3.1）。
5. GC 后仍被 trace 引用的根消息必须保留、链可完整回放（M10 Task 10.2）。
