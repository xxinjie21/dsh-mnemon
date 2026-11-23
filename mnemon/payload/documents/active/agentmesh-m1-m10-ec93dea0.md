---
id: "ec93dea0-9945-4c6a-86a3-2cff7e2b352b"
title: "AgentMesh 实现落地：M1–M10 完成态、运行入口与设计边界"
description: "AgentMesh 从 PRD/mock 走到可运行工作台后的实现落点：M1–M10 模块与单测映射、npm start 运行入口、通知/去重/GC/诊断包四条必须保住的设计边界、Gate-0~4 真实状态与用户要求的汇报口径。"
status: "active"
created_at: "2026-10-29T16:27:41.107Z"
updated_at: "2026-10-29T16:27:41.107Z"
content_hash: "d8c4570ccb63c766194b97d88c356ac96f908f54f4014fe0662532111409ec6c"
source_paths:
  - "src/core/notices.js"
  - "src/core/artifacts.js"
  - "src/core/health.js"
  - "src/core/gc.js"
  - "docs/evidence/gate-3.md"
  - "docs/evidence/gate-4.md"
  - "test/unit/m8.test.js"
  - "test/unit/m9.test.js"
  - "test/unit/m10.test.js"
  - "test/unit/gate4-close.test.js"
session_ids:
  - "b284af8b-e6ac-443d-b7ad-0b28ee81fd7d"
memory_body_ids:
  []
---

# AgentMesh 实现落地：M1–M10 完成态与运行入口（替代“仅 PRD + mock UI”基线）

> 来源：工作区 `D:\ZISHIKU\AI\AgentMesh` 的已完成实现会话（提交 124a9ff / ddc94ce / 2cec48c / 8e916fb / 0ff4631）。相关文档：`74ed2e47-2a1d-4598-a22a-a03090a8d351`（PRD 4.1 交接要点，其“仓库仅有 mock UI”的现状描述已被本文取代）、`cdd9b45a-3b0f-4950-9924-71bb5628b1f4`（PRD 2.7→3.0 架构与计划）。本文只记实现结论、边界理由与运行方式，不复制 PRD 正文。

## 1. 现在是什么

AgentMesh 是一个**本机多 Agent / 多应用工作台**：Node.js + SQLite 单文件库（`data/agentmesh.db`，schema_version=10），只绑 127.0.0.1，浏览器界面 + REST/WS。它只做消息传输与转发，**不替用户编排顺序**，自身不调用模型。

最短使用路径：`npm install` → `npm start` → 打开 `http://127.0.0.1:7600/` → 界面点「连接本机」加载真实数据（不再有 `?demo=1` 静态 mock）。尚未做安装包 / 桌面打包，也没有对 OpenCode 之外的其他 CLI 逐个接入。

## 2. 里程碑实现落点

| 里程碑 | 关键实现 | 单测 |
|---|---|---|
| M1 Herdr 连通 | socket 接入、pane 开关/读/写/agent start；真机 e2e 已过 | — |
| M2 适配器 | CLI(herdr) / GUI(WS) / MCP 三类，四级接入分级 linked/connected/hosted/none | — |
| M3 消息闭环 | trace_id、投递队列与 seq、单播/组播 | — |
| M4 小组/子代理 | 小组组播；`spawnChild` = paneSplit + agentStart，最多 3 层，可自动关闭与级联销毁 | — |
| M5 派活 | 一句话派活、10 秒内撤回、模板“沿用上次设置” | — |
| M6 看板 | 进行中/等待中/已完成/失败；WS 推送 1.2 秒内刷新 | — |
| M7 失败重来 | `src/core/` 失败标定 + retry 只重发失败那一步；Herdr 故障时托管节点 offline 而 GUI/总线不受影响 | `test/unit/m7.test.js` |
| M8 待我处理 | `src/core/notices.js`：assigned / waiting_on_you / failed 三类；同 `merge_key` 未归档则更新原通知；`actor === target` 不通知自己；四档偏好（全部/只看失败/只看找我/不提醒）推送时过滤；通知只带一句摘要 + `ref_msg_id`，不携带正文；归档只改通知状态不动任务；`/api/notices/{id}/answer` 走真实 reply；请求超时 → 任务 failed + 一条失败通知 | `test/unit/m8.test.js` |
| M9 省用 | `src/core/artifacts.js`：registerArtifact(file/output/summary) + `GET /api/artifacts?task_id=`，任务详情附带 artifacts；同一 trace 内内容哈希重复 → 标记 duplicate 并指向首次 msg_id，**原始消息不删**（展示层去重）；看板显示“N 条消息 · M 字符”；>40 字摘要默认折叠可展开 | `test/unit/m9.test.js` |
| M10 运维 | `src/core/health.js`：`GET /api/health` 免 token、毫秒返回 Core/Herdr/DB/接入分级 + 自愈动作（自动补投 vs 人工重启）；`POST /api/diagnostics` 输出元数据，断言不含 token 明文/哈希/正文；`src/core/gc.js` `runGc` 删过期消息、**保留 trace 根**并清理投递/引用，REST `POST /api/gc` 与 CLI `agentmesh gc` 共用实现 | `test/unit/m10.test.js` |
| Gate-4 收尾 | 20 节点 + 200 条消息看板读取 107ms（NF2 ≤2s）；待送达达 1000 条只提示一次 `warning.over_limit` 且照样入队；断线期间 pending 保留、`pumpNode` 重连后按 seq 补投；`runGc` 幂等 | `test/unit/gate4-close.test.js` |

测试总量：105 tests / 103 pass / 0 fail / 2 skip（`npm test` 全量）。

## 3. 设计边界（改动时必须保住）

- **通知不是消息副本**：摘要 + 合并键 + 偏好过滤 + 归档隔离；归档不影响任务状态。
- **去重不删消息**：同链同内容只“少搬”，不是“少存”；原始消息必须仍可回放。
- **GC 不删 trace 根**：否则复盘材料消失；过期普通消息才回收。
- **诊断包自检**：返回前断言无 token 明文/哈希/正文，避免把凭据带进导出物。
- **消息表只追加**：触发器禁止 UPDATE，测试里不能用改时间戳的方式模拟历史数据，需推进时钟（此前 m10 GC 测试即因误用 openDb 而报 `messages is append-only`）。
- **不擅自停机演练**：真机停 Herdr → 恢复的端到端演练会中断用户正在使用的 Herdr 会话，需用户同意才做；投递层语义已由单测固定。

## 4. Gate 状态（据实）

- Gate-0 / Gate-1：已关闭。
- Gate-2 / Gate-3 / Gate-4：证据写入 `docs/evidence/gate-2.md`、`gate-3.md`、`gate-4.md`（含 `gate-4-ui.png`）。
- 仍未做：真机 Herdr 重启端到端演练（可选、需用户批准）；诊断包用 JSON 未打 zip（少一个 `yazl` 依赖）；录屏以截图 + 接口证据代替。

## 5. 协作口径（用户明确要求）

用户在会话中明确抱怨“老是提 Herdr”，并要求不再把“待批准的真机演练”挂在每次交付末尾。因此：重复未做项不作为交付模板尾巴；未决项只写进各 gate 证据文件，用户主动问才复述。同时用户指出“做了什么软件”被里程碑汇报掩盖——对外说明应先讲“这是个能打开来用的工作台”，再提验收细节。
