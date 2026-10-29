---
id: "bfb7992c-61af-46a4-91e7-df08c6495f4b"
title: "DSH 上下文自动压缩调查与 smart-compaction 落地：内核 compaction 事实、插件选型证据与配置写法"
description: "DSH 内核 dsh-compaction-basic 自动压缩存在性证据、触发点偏晚的根因、smart-compaction v0.3.12 源码要点与 safePointRatio 配置覆盖写法、asar header 读取踩坑与插件选型活跃度对照。"
status: "active"
created_at: "2026-10-19T17:16:04.504Z"
updated_at: "2026-10-19T17:16:04.504Z"
content_hash: "6f919de02d0560677c45d57998ff7639c9971f00fab4c549542a70ddbe6a28c9"
source_paths:
  []
session_ids:
  - "420cd183-d660-4a57-8fe2-741da9bb2662"
memory_body_ids:
  []
---

# DSH 上下文自动压缩调查与 smart-compaction 落地

调查触发点：用户认为 DSH 不会自动压缩上下文、对话变长不主动精简。本次核实结论如下（2026-10-20）。

## 结论先行

1. **DSH 内核本来就内置自动压缩**，不是"完全不压缩"。安装包中存在 `dsh-compaction-basic`，asar 内 `compaction` 相关标识符命中 2071 次（`compactionId` 184、`compactionRetries` 19、`compactionButton` 14、`compactionFailed` 3 等），另有 `compactions`、`compactionSummary` 等 UI/状态字段。该会话此前确实发生过一次压缩（存有 compress 记录）。
2. 真正的体验问题是**触发点偏晚且不可控**：压缩由固定 token 阈值驱动，模型无法看见真实用量、也无法主动触发。用户接入的百炼/中转模型声明 `contextWindow: 1000000`，但这是**声明值**，与实际有效窗口未必一致。
3. 落地方案：装 `smart-compaction` v0.3.12，提供 `context_status`（真实用量 vs 窗口）与 `compact_now`（回合中途主动压缩），并带 `safePointRatio` 自动触发（默认 0.5）。

## 环境事实（本次核实）

- DSH 以 Electron 分发，内核在 `D:\DeepSeek Harness\resources\app.asar`（174.3 MB）。`@deepseek-ai/dsh-base` **不在** profile 的 `node_modules` 里——它是内核内置模块，由 Electron 运行时提供，因此 `dsh-compaction-basic` 也在 asar 内。
- 本 profile bundles（`C:\Users\xxinj\.dsh\profiles\desktop`）不单独列出任何压缩包：`@deepseek-ai/dsh-base`、`@deepseek-ai/dsh-web-app` 之外没有任何 compaction 包。压缩能力打包在 dsh-base / dsh-web-app 内部。
- 当前模型配置：`@deepseek-ai/dsh-agent-default-model`，provider `qwen-token-plan-cn`，model `deepseek-v4-flash-0731`；`contextWindow: 1000000`（三处出现）。
- 内核中**没有**可配置的 autoCompact / compactThreshold 开关：对 `autoCompact`、`auto_compact`、`compactThreshold`、`compressTurn` 的关键词扫描命中均为 0；`mnemon-strategy-light-context` 中也无相关配置项。

## 插件市场盘点（3036 个插件中相关项）

| 插件 | 作用 | 核实活跃度（2026-10-20） |
|---|---|---|
| `@indexyz/dsh-autocompact` | 按 provider/model 单独设置更早的自动压缩阈值，`/autocompact` 写 session-only override | 3★，repo 5aaee9/pi-agent-extensions，push 2026-10-18 |
| `smart-compaction` | `compact_now` + `context_status`，模型可主动压缩 | 1★，JoblessJoe/smart-compaction，push 2026-10-16 |
| `dsh-compact-selected-model` | 修 `/compact` 与自动压缩使用"上一回合完成的模型"而非当前会话选中模型的缺陷 | 0★，DenAbr，push 2026-10-11 |
| `@goodandready/dsh-context-lens` | AST 级压缩 + 日志过滤 + token 预算护栏 | 4★，GooDAnDReaDY，push 2026-10-19 |
| `dsh-idle-compact` | 空闲超时后、KV cache 过期前压缩 | 0★，Arrosam，push 2026-10-19 |
| `@internetnutzer/dsh-asc` | 模型驱动的压缩，代理自行决定压什么 | 0★，JanEickholt，push 2026-10-17 |

风险提示：以上全是 0–4 star 的个人项目，会改变压缩阈值与行为，一旦有 bug 可能导致压缩时机错乱或摘要丢失。选型建议是**先只装 smart-compaction**（只增加"看得见 + 可主动触发"），确认稳定后再考虑 autocompact。

## smart-compaction 实现要点（v0.3.12，已读源码）

安装位置：`C:\Users\xxinj\.dsh\profiles\desktop\node_modules\smart-compaction`，登记在 profile 的 bundles 与 deps（`smart-compaction=0.3.12`），mtime 2026/10/20 1:14。

- 包结构：`index.js`（11589 B）、`context-status.js`、`safe-point.js`、`select-range.js`、`cordis.patch.yml`、README.md。`package.json` 中 `dsh.bundle.patch: ./cordis.patch.yml`，兼容性 `dsh >= 0.2.0-rc.2`。
- `cordis.patch.yml` 挂载 id 为 `tool-compact-now`，**无需任何配置**（"nothing to configure"），一个模块同时注册两个工具。
- 调用 `ctx.compaction.compactRegion()` 而**非** `compactNow()`：工具的 `execute()` 总在回合内运行，`compactNow()` 会因 `busy` 每次抛错。
- `compaction` **故意不放进 `inject`**：否则在没有压缩后端的 profile（如 dsh-web-app）上会启动失败；改为调用时 `ctx.get('compaction') ?? ctx.get('agentPresets')?.serviceFor(agent, 'compaction')`。
- "正在压缩"提示在慢调用**之前**以 `user/message` + `source.kind: 'smart-compaction'`、`form: 'notice'` 发布，避免聊天看起来卡死。
- 用 `selectCompactableRange` + `toolPairingBalancedBefore` 选择可压缩区间，保证只在工具调用对完整处切开。
- `context_status` 通过 `sessionProjections.stateOf(session, 'contextPressure')` 取真实用量；无 projection 时返回 `{available:false}`。
- **自动触发**：`safePointRatio` 默认 0.5，即用量达 50% 且刚完成一个 todo 项时自动压缩；`false` 可关闭。
- 失败分支：若无压缩服务会抛 `compact_now: no compaction service is configured for this agent`。

## 生效前提

插件工具在会话启动时注册，本回合工具集已固定，因此装完必须**重启 DSH** 后 `compact_now` / `context_status` 才出现。

## 可选的 profile 覆盖写法

若觉得 0.5 对 1M 窗口太早（= 50 万 token 就压，压缩本身要一次额外模型调用），在 profile `cordis.patch.yml` 中覆盖：

```yaml
- id: tool-compact-now
  name: smart-compaction
  config:
    safePointRatio: 0.7   # 或 false 关掉自动触发
```

## 排查方法与踩坑（可复用）

- 直接读 asar header 的正确偏移：offset 0 为 pickle 长度 `04 00 00 00`，offset 4 为 header 长度（`0x48EF0C`），JSON 从 **offset 16** 开始（`7B 22 66 69 6C 65 73` = `{"files"`）。
- 本次多次用错 header 长度导致大输出浪费并报 JSON 尾部错误；一旦连续两次失败应停止在 asar 上试错，改用插件市场元数据或已成功过的脚本公式（`8 + headerSize` 前的 pickle 解析方式）。
- 判断某项内核能力是否存在，先扫 asar 的标识符频次（能确认存在），再查 profile bundles 判断实现归属；无独立包名通常意味着内嵌在 dsh-base / dsh-web-app。
