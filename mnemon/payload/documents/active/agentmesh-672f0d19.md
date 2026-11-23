---
id: "672f0d19-ab57-44fc-892f-4224d5dfb5b9"
title: "AgentMesh 硬化范围收敛（本地单用户）+ 原型前端接入边界"
description: "硬化计划从商业口径收敛为本地单用户可交付（保留 A–E，砍掉多租户/RBAC/原生 GUI/安装备份/24h 长稳），以及原型 HTML 必须改本体、暂不接后端、禁止桥接副本的返工结论与环境事实。"
status: "active"
created_at: "2026-10-30T04:51:19.053Z"
updated_at: "2026-10-30T04:51:19.053Z"
content_hash: "860942d714a7e6d3ab51a63aa99101ce20a7779dc2a22a1011d55bec882cf508"
source_paths:
  - "docs/superpowers/plans/2026-10-30-commercial-hardening.md"
  - "prototype/AgentMesh-原型.html"
  - "src/api/http.js"
session_ids:
  - "aead4319-bdda-4874-8f61-41331a4299ab"
memory_body_ids:
  []
---

# AgentMesh 硬化范围收敛（本地单用户）+ 原型前端接入边界

> 工作区 `D:\ZISHIKU\AI\AgentMesh`。2026-10-30 一次范围收敛 + 一次前端接入返工后的结论。相关文档：硬化计划总览 `31c40d8c-76a0-41e4-a1bb-7e710c61711e`（A–F / G0–G6，商业口径），M1–M10 骨架 `938fa791-fc48-45e8-9a87-8a67b2a6ab92`。本文只记录收敛后的边界、理由和前端协作方式，不复述计划正文。

## 1. 范围收敛：不是企业协作平台

产品目的被重新钉死为**本地单用户多 Agent / 应用通信工作台**，而不是 SaaS / 多租户商业产品。原计划把“商业可用”扩成多租户、RBAC、原生 GUI 嵌入、安装升级备份、24h 长稳，被认为成本失控，因此按“能演示 → 本地单用户可交付”重写。

保留（直接服务主链路）：
- A 真实 Agent 发现、MCP 配置、握手、结构化回复、分步降级；
- B terminald、Core 重启 reattach、进程监管、孤儿回收；
- C 幂等、重试、死信、重连、取消、背压；
- D 本地 token、最小审计（`audit_events`）、可选通信线强制（`link_enforced` 默认关闭）；
- E 关键故障注入、2h soak、10k 队列压测；
- 贯穿原则：零模型 token 控制面、错误可见、可恢复、可诊断。

明确不做（写入计划正文的“不做”清单，避免以后被重新拉回）：
- 多租户 / 工作区 / 团队 RBAC / 策略引擎 / 企业 SSO / 计费 / 云端部署；
- 原生 GUI 窗口嵌入（窗口坞只是占位，不对外宣称已原生嵌入）；
- 安装包、自动更新、商业备份产品；
- 24h 企业级长稳（降为 2h soak + 10k 队列）；
- 自动摘要、自动复盘、默认 LLM 调用。

口径变化：完成标准从“商业可用”改为“本地单用户可交付”；G4 从权限矩阵收敛为“本地安全 + 最小审计”；G5 从 24h 收敛为 2h soak + 10k 队列；数据侧新增 `audit_events`，`links` 增加本地可选的强制标记，不引入 workspace/role/RBAC 表与接口。对外表述边界：在 Gate 达成前只能说“本地工作台原型”。

理由与风险：`link_enforced` 可能打断现有流程，所以默认关闭、先审计不拦截再灰度；多租户属于过度设计；GUI 嵌入各平台差异大（Windows SetParent/DWM、macOS 多为外部窗口捕获、Wayland 受限），本轮不做。

落点：`docs/superpowers/plans/2026-10-30-commercial-hardening.md`（标题改为「核心功能硬化计划（本地单用户版）」），提交 `d436e73 docs: right-size hardening plan for local single-user scope`。

## 2. 原型前端：改本体，不做桥接副本

用户指定前端参考物是 `prototype/AgentMesh-原型.html`（file:// 绝对路径），要求：**不参考项目自带前端，把后端能力迁到该原型上；不要大改该文件，只做协调**。

第一次做法（复制成 `ui/prototype.html`、加 `window.__AM` 暴露、加 `ui/prototype-bridge.js`、把 `/` 指到副本）被明确否决：用户要的是改原型文件本体，而且**当前阶段先不接后端**——先把前端交互写完整，再反推需要哪些后端接口。

纠正后的状态（提交 `160cd20 chore(frontend): serve standalone prototype without backend bridge`）：
- 删除副本与 `ui/prototype-bridge.js`，原型文件恢复为与备份一致，不残留 bridge / 暴露脚本；
- 后端只保留一条静态路由：`/` → `prototype/AgentMesh-原型.html` 原文件，`/classic` → 旧界面；无桥接逻辑；
- 原型独立运行，仍是前端 mock，交互不接后端。

后续协作约定：只改 `prototype/AgentMesh-原型.html` 本体；先补前端交互（新建会话、发送、暂停/继续、交接、产物、失败重试、搜索/归档），等前端行为定稿再反推后端接口；不擅自加 bridge、不复制前端、不改版式。

## 3. 环境事实

本机未安装 ffmpeg / ffprobe，无法从 mp4 抽帧；模型侧也没有原生视频输入，视频类需求只能通过关键帧截图分析。当前 HTTP 服务监听 `http://127.0.0.1:7600`（后台任务，可能随会话终止）。
