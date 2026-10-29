---
id: "d920cff7-e01e-4749-b18e-89dfadb6e2f3"
title: "DSH 推理模型（deepseek-v4.1-flash）用尽 max_tokens 导致正文为空、\"额度已用尽\"显示 0%：根因、实测证据与 enable_thinking 取舍"
description: "DSH deepseek-v4.1-flash \"额度已用尽\"根因：推理与正文共用 max_tokens 配额导致正文为空；百炼忽略 thinking_budget，enable_thinking:false 是唯一实测可行方案，附 85% 压缩点取舍与内核 thinkingFormat 配置 API。"
status: "active"
created_at: "2026-10-20T04:57:15.277Z"
updated_at: "2026-10-20T04:57:15.277Z"
content_hash: "d6ff556acc5e29380d58126409d73892f2d58b824468a0c0e8d5bc93aca720b4"
source_paths:
  []
session_ids:
  - "87a6d893-95f2-4f83-b22e-0780634c72eb"
memory_body_ids:
  []
---

# DSH 推理模型额度归零：根因、实测证据与配置取舍

## 现象

DSH 中 `qwen-token-plan-cn` / `deepseek-v4.1-flash` 在复杂任务上偶发"额度已用尽"、上下文用量显示 0%；切换模型再切回即恢复。用户目标压缩点按 **85%**（`safePointRatio` 见 `bfb7992c-61af-46a4-91e7-df08c6495f4b`），为给压缩留余量已把 `maxTokens` 从 384000 降到 **32768**。

## 根因（已决定性复现）

`deepseek-v4.1-flash` 是推理模型，**思考过程与正文共用同一个 `max_tokens` 配额**。遇到需要长思考的任务，小配额被思考全部占满 → 正文返回空字符串 → `finish_reason: length` → DSH 走异常分支显示"额度已用尽"，用量投影归零（显示 0%）。切模型会重建 client 状态，故切回来恢复正常。

决定性实测（同一道复杂推理题，`mt` = `max_tokens`）：

| mt | finish | reasoning tokens | 正文 tokens | 正文为空 |
|---|---|---|---|---|
| 128 | length | 128 | 0 | 是 |
| 512 | length | 512 | 0 | 是 |
| 2048 | length | 2048 | 0 | 是 |
| 32768 | stop | 16539 | 1097 | 否 |

同一题在 `enable_thinking: false` 下：`mt=128 → 正文 128`、`mt=512 → 正文 512`、`mt=32768 → 正文 1178`，`reasoning` 均为 `-`，延迟 2057ms → **701ms**。

## 内核 provider 配置 API（在 `app.asar/dsh` 内核代码中核实）

- `thinkingFormat`：`'openai' | 'openrouter' | 'deepseek' | 'together' | 'baseten' | 'zai' | 'qwen' | 'chat-template' | 'qwen-chat-template' | 'string-thinking' | 'ant-ling'`（默认 `openai`）。**`'qwen'` 模式内核会自动发送 `enable_thinking`**，值取 `reasoningEffort`。
- `chatTemplateKwargs`（配 `thinkingFormat: 'chat-template'/`'qwen-chat-template'`）与 `chatTemplateArgs`（配 `'baseten'`）：值可用 `$var` 占位，合法取值只有 `thinking.enabled` / `thinking.effort` / `thinking.budget`（内核 `CHAT_TEMPLATE_VARS` 白名单，共 3 个）。占位对象支持 `{ '$var': 'thinking.budget', omitWhenOff?: boolean }`。
- `thinkingTokenBudgetField: 'thinking_token_budget' | 'thinking_budget' | 'thinking_budget_tokens'` 与别名 `supportsThinkingTokenBudget`：Top-level 字段，用于把思考 token 预算下发给 `thinkingBudgets`。源码注释明确标注面向 **vLLM / Qwen / llama.cpp**，且**默认关闭（Off by default）**。另有 `vLLM`/Qwen 的 `vllmPriority` 等。

## "给思考单独设预算"为何走不通

百炼（`dashscope.aliyuncs.com`）实测三种字段名全部被静默丢弃，行为与无预算完全一致：

```
thinking_budget=200        → reason=512  text=0  empty=true
thinking_budget_tokens=200 → reason=512  text=0  empty=true
thinking_token_budget=200  → reason=512  text=0  empty=true
enable_thinking+budget     → reason=512  text=0  empty=true
```

因此无法在百炼上只压制思考预算；该路径只对自建 vLLM/Qwen/llama.cpp 网关有效。

## 方案与取舍

| 方案 | 实测效果 | 代价 |
|---|---|---|
| A. `enable_thinking: false`（仅对推理模型） | 正文正常返回，小 `maxTokens` 也不空，延迟 2057ms→701ms | 失去推理模型的思考能力 |
| B. 提高 `maxTokens` | ≥32768 才够该题，更复杂题仍可能耗尽 | 压缩触发点被 `W−O−B` 拉回，85% 目标作废（提到 65536 约回落到 87%） |

建议 **方案 A**：保持 `maxTokens=32768`，压缩点仍落在 85%。

## 未验证风险（落地前须知）

`llm-pi-ai` 的自定义 provider 是否接受 `thinkingFormat` / 关闭思考的配置字段**尚未验证**，需实配一次并抓包确认 `enable_thinking` 确实以 `false` 发出。若配置不被接受，应明确说明而非假定成功。

## 教训（可复用）

- 推理模型在小 `max_tokens` 下返回**空正文 + `finish_reason: length`** 是共用量配额所致，不要误判为服务端额度/鉴权问题。
- 排查此类问题应同时打印 `reasoning tokens`、正文长度与 `finish_reason`，缺一项就会得出错误结论。
- Windows PowerShell 会把中文引号当字符串结束符；写探针脚本请落成脚本文件再执行，避免内联解析问题。
