---
id: "d373d32b-0b73-4671-8066-ccf944a194b2"
title: "AgentMesh 一期设计定稿：本地多 Agent／应用工作台、send-request 消息原语与 Herdr sidecar"
description: "AgentMesh 项目一期定稿：强定位（multica 缩小版 + Herdr 扩充版）、三路由、send/request 两种消息原语、Herdr 独立 sidecar、SQLite+Trace、已否决方案、实现前 spike 与 12 个对标产品调研结论。"
status: "active"
created_at: "2026-10-28T14:12:14.201Z"
updated_at: "2026-10-28T14:12:14.201Z"
content_hash: "5eb525cd9771274c6de1208e4f9a4258f82e28fd312da5ab768d692bf940488f"
source_paths:
  - "AgentMesh.md"
session_ids:
  - "711a9410-bd88-40c1-b177-34e6a054384b"
memory_body_ids:
  []
---

# AgentMesh：本地多 Agent / 应用工作台（一期设计定稿）

> 来源：`AgentMesh.md`（工作区 D:\ZISHIKU\AI\AgentMesh，996 行 / 48,315 bytes）。项目按用户要求只维护这一份文档：目的、一期设计、调研依据、交付记录四部分合一；旧的目的说明、定稿方案、一期设计 V2、独立交付记录已合并删除。

## 1. 定位与判断准绳

- 一句话：本地多 Agent / 应用工作台——把多个 CLI Agent 与 GUI 应用统一接入，让用户自定义它们之间的通讯与协作。
- 强定位：**multica 的缩小版 + Herdr 的扩充版**。借 multica 的"消息总线 + 节点协作"思想但只做必要功能、不内置固定角色流水线；把 Herdr 从单终端 PTY 托管扩充到多节点协调调度，保留其原生能力。
- 判断准绳：强定位优先于任何局部设计；9 点意图只定思想和目的，不固定实现（schema、API、UI、Herdr、Spawn 细节可按目标调整）。单机本地、SQLite、可复盘、可重试是底线。
- 项目只有一期：真实 CLI 与真实 GUI 都在本期接入，不设二期、不做预留式交付；模拟节点只是开发期联调手段，不是独立阶段。

## 2. 三条通信路由（用户自定义，无规则引擎）

| 路由 | 语义 | 要点 |
|---|---|---|
| Group | 用户建组、加成员、组内广播 | 一个节点可入多个 Group；适合广播型协作 |
| Spawn | 已有节点派生 CLI 子代理 | 完整树模型，一树一 trace；scope = parent / children / tree；子节点必须是 CLI Agent；支持级联销毁 |
| 点对点 | 两个节点直连 | 需要等待/回复时用 request |

关键取舍：**自定义取 A 不取 B**——用户在三种固定模式里选，并决定谁进哪个组、谁派生谁、谁和谁私聊；不做用户可编程的路由规则引擎。

## 3. 消息原语：只有 send 与 request 两种模式

这是本项目最重要的一次设计收敛（替代早先"expects_reply 参数 + 第三种行为"的方案）：

| 模式 | 语义 | 调用方行为 |
|---|---|---|
| send | 单向发送，不等回复 | 发完继续做自己的事 |
| request | 请求-应答，带 correlation_id | 自己选择立即等待，或先并行、稍后取回复 |

边界必须守住：

- **wait / parallel 由调用方 Agent 决定，不塞进路由层**；总线只做关联、路由、状态记录。
- `reply` 是 request 的响应，**不是第三种模式**。
- OpenCode、WorkBuddy 等只是**调用者**，不是总线里硬编码的流水线节点；不执行任务、不强制谁先谁后。
- Group / 树用 send；需要等待就用点对点 request。

## 4. 节点、进程与接入

- **CLI Agent**：经 Herdr pane 接入，保留原生终端、工具、权限、模型。
- **GUI 应用**：取"接入"而非"托管"——应用自己运行，通过适配器（参考实现为 WS 适配器）收发消息；AgentMesh 不启动、不杀进程、不嵌窗口。
- **Herdr 集成**：独立 sidecar，Core 通过 Herdr socket API 管理 pane（pane.split / pane.close / pane.read / agent.prompt / pane.report_agent）；不 fork、不做插件，因而保留 Herdr 原生能力。
- 本地安全：只监听 127.0.0.1，每节点 token；不做远程。

## 5. 消息、Trace 与持久化

- 消息元数据：`trace_id`（任务分支）、`ref_msg_id`（父消息）、`retry_of_msg_id`（重试谱系）。
- 重试语义：**只重发消息，不自动重跑 Agent**——传输重试与业务重跑分离。
- 持久化：SQLite（WAL + 单写者 + busy_timeout），存节点、Group、Spawn 树、消息、请求、投递记录、模板；带 retention 字段与 GC 命令。
- 人机协作：`waiting` 作为可选状态纳入；**审批拦截不纳入**（审批是 Agent/应用原生能力）。
- `permission_mode` 不纳入设计（权限属于 Agent 原生能力）。

## 6. 已否决 / 已收敛的方案

- 不做用户自定义路由规则引擎；三路由固定。
- 不托管 GUI 应用（不启动、不嵌窗）。
- 不 fork Herdr，也不做 Herdr 插件。
- Spawn 不做"只子→父"，做完整树。
- 不做硬编码流水线（例如把 OpenCode→WorkBuddy 写死成一条链）。
- 不设二期。
- 评分按用户要求不记录（交付记录中的历次"评分"小节均为空声明）。

## 7. 实现前必须过的四个 Spike

1. Herdr pane.split / pane.close / pane.read / agent.prompt 最小闭环；
2. MCP server 与 Core 的握手、收发、ack；
3. WS GUI 适配器的注册、心跳、消息往返；
4. SQLite 重启对账与 Herdr pane 映射恢复。

主要风险与对策：Herdr socket API 版本差异 → 启动时执行 `herdr api schema --json` 校验；CLI 不支持 MCP → 先选一个支持的做样板，其余标"未接入"；GUI 适配器缺失 → 先做参考 WS 适配器，协议写清由应用侧实现；SQLite 并发写 → WAL + 单写者 + busy_timeout，压测 20 节点；终端串流延迟 → pane.read 事件批量推送，终端只展示不承载消息；范围膨胀 → 以一期验收条款为准。

## 8. 调研依据（12 个对标产品）

multica（Issue 评论 @ 触发 run、squad 路由）、OpenCode（文件即 Agent、子代理、权限规则）、DeepSeek Harness（Cordis 插件化、append-only 会话日志 + Trajectory，UI 直接对标对象）、Orca（多 worktree 并行舰队）、Claude Code（Markdown+YAML 子代理声明）、Cline（子代理可读写、shadow git checkpoint 回滚）、Gemini CLI（Policy Engine TOML、硬禁递归）、GitHub Copilot（云端自治 Agent + 并行舰队）、Bolt.new（Standard/Max 两档 build agent）、Gumloop（节点画布 + 字段级引用）、n8n（运行状态机、Wait/Webhook 长时暂停）、Flowise（LangChain 可视化编排、SQLite checkpoints）。

结论：定位成立——"多 Agent/多应用统一工作台 + 消息总线 + 可复盘 trace"区别于重编排平台。两条重要纠偏：Bolt 不是 planner/builder/runner 流水线；Cline 子代理不是只读，Act 模式可读写、可执行命令。调研事实会过期，接入前需按官方文档复核目标产品版本。

## 9. 交付形态结论

- 单一项目文档优于多文档各自演进：读者不必来回对照，冲突处只有一个口径。
- 历史交付记录（含 PUA 输出）保留在文档第四部分，不丢；评分不记录。
- 已知遗留：文档第三部分曾出现重复标题"## 1. 对标产品"与重复的 DeepSeek Harness 来源行（合并后已修正，最终版 996 行）。
