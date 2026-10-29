---
id: "76fb4016-6cdd-4f97-9f1e-3e2fd13e9c98"
title: "tcm-ehr-qc 清洗异步化与「提交后立即反馈」修复闭环（Governance 前端）"
description: "清洗异步任务化、刷新恢复运行中任务、提交后立即显示执行中的两轮根因（预览接口 await 阻塞）、端口归属诊断法与分批提交纪律、本轮 git 现场与待办。"
status: "active"
created_at: "2026-10-20T17:48:41.974Z"
updated_at: "2026-10-20T17:48:41.974Z"
content_hash: "7d9fbb57d855a8183e90916f4a3cc03dc66bcb65b18e498f7fc997773711ac13"
source_paths:
  - "frontend/src/views/Governance.vue"
  - "frontend/src/api/governance.js"
  - "frontend/src/views/Qc.vue"
session_ids:
  - "c43c1ece-0da5-4907-ba25-2f083030d04c"
memory_body_ids:
  []
---

# tcm-ehr-qc 清洗异步化与「提交后立即反馈」修复闭环（Governance 前端）

## 背景
数据清洗原为同步接口，长任务体验差。本轮（2026-10-21）在 Governance 页完成三项 UX 交付，并修复了两次未生效的回归。

## 交付与修复要点
1. **异步化 + 进度轮询**：清洗改为异步任务（提交即返回 taskId），前端轮询 `GET /api/governance/clean/batch`（单任务状态）与 `GET /api/governance/clean/batch` 列表接口（恢复运行中任务）。进度 percent = done/total，终态集合 COMPLETED/CANCELLED/INTERRUPTED/FAILED。
2. **刷新/切页恢复运行中任务**：`onMounted` 调用 `restoreRunningClean()`，从任务列表找 QUEUED/RUNNING 的任务接回轮询；`doNormalize` 由任务 `skipNormalize` 反推恢复。
3. **提交后立即显示「清洗执行中」**（两轮才修对的关键点）：
   - 第一次只改了「提交成功后置 loading」，未生效。
   - **真正根因**：确认弹窗后先 `await previewDone`（预览接口实测 6~7 秒）才置 `clean.loading = true`，用户点确认后这 6~7 秒按钮毫无反应。
   - 修法：点击确认后**立即**置 `clean.loading = true`，把预览等待挪到后面（此时已有视觉反馈）；`submitCleanBatch` 成功后再置一次 `loading=true / percent=0` 作防御，随后 `startCleanPoll()`。
   - 轮询体抽出为 `pollCleanOnce`，`startCleanPoll` **先立即拉一次**再 `setInterval(2000)`，避免提交后空等一个周期才出现 percent。

## 诊断经验（可复用）
- **「改了没生效」先查服务身份**：本机 :3000 是 Vite dev server（读源码 + HMR），:80 实际由 Steam++ Accelerator 占用（无关进程）。构建到 `frontend/dist/` 只对静态托管生效，dev server 页面看的是源码。用 `Get-CimInstance Win32_Process` 查 CommandLine 判定端口归属，不要凭端口号臆测。
- **前端「无反应」类问题优先查 await 链上的隐藏长耗时调用**（这里是预览接口），而不是只查状态置位语句本身。
- 用户回报"没实现成功"时先复现/定位根因再改，避免反复猜测式修改。

## 提交纪律（用户明确要求）
- 用户要求「分批次提交未提交的文件，然后推送」。
- **禁止 `git add -A`**：只提交本轮有意改动的文件；无关改动（如他人改的 `Qc.vue` 的 max-height→height、历史 docs 删除/修改）一律跳过。
- 用 ops 工具产生的 `.dsh-ops-update-*.lock` 残留需清理，不应入库。

## 本轮 git 现场
- `404b99e` 修复清洗进度条不动 + 操作区排版重排
- `c455db5` 刷新/切页后恢复运行中清洗任务 + 提交后立即显示执行中（2 files changed, 51 insertions）
- `a2c1cd3` 确认清洗后立即显示「清洗执行中」，不再被预览接口阻塞（1 file changed, 7 insertions, 4 deletions）
- `bfd74d4` docs: 补词候选清单（症状第三批 / 高频 2000 / 修正版高频 3000，3 个 CSV，8031 insertions）
- 推送成功：`e7ff066..bfd74d4  main -> main`（远端 https://github.com/xxinjie21/tcm-ehr-qc.git）。
  推送时 git-credential-manager 路径报 "No such file or directory" 但推送仍成功——该报错可忽略。

## 仍待办
- 豆包那批证候/方剂（+20 标准词）已落库并重建 ES，但**仅归一尚未跑**（约 8 分钟），跑完才能回传真实归一率。
- 工作区剩余无关改动（docs 删除/修改、`Qc.vue`）未提交，等待用户决定。

## 来源
整理自 DSH 委托维护检查点的工具输出（`Governance.vue` 与 `governance.js` 片段、构建输出、端口/进程查询、git status/diff/log/push 现场），非本轮新查询。仓库根 `D:\ZISHIKU\AI\tcm-ehr-qc`。

## 关联文档
- `e2138429-3421-48b9-b77f-ccc47c387b6a`：tcm-ehr-qc 全库重抽闭环与归一率口径辨析（清洗/归一链路上游结论）。
