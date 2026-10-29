---
id: "2b6eba09-014a-4c5b-ac9b-8b793f96bcaf"
title: "DSH 对 glm-5.3 自动发送 enable_thinking=false 导致百炼 400：根因与 reasoning:false 修法"
description: "百炼自定义 glm-5.3 在 DSH 报 400 \"enable_thinking 只能为 True\" 的根因：内核 thinkingFormat=qwen 且 model.reasoning 为真时无条件发送 enable_thinking=!!reasoningEffort；修法是在 profile 模型条目加 reasoning: false，附实测证据与 deepseek-v4-flash-1017 名称不存在问题。"
status: "active"
created_at: "2026-10-21T06:22:36.147Z"
updated_at: "2026-10-21T06:22:36.147Z"
content_hash: "e669f53c7c3aee2e15a3b2ce8f1b2f3d28ec7c9e6d3c69f43d4f3c97e9cf064e"
source_paths:
  - ".dsh/profiles/desktop/cordis.patch.yml"
  - ".config/opencode/opencode.json"
session_ids:
  - "b5333b97-ad06-4eda-a41a-55ea3cb25869"
memory_body_ids:
  []
---

# DSH 对 glm-5.3 发送 `enable_thinking=false` 导致百炼 400

## 症状

在 DSH 中使用百炼上的 `glm-5.3`（自定义 provider `qwen-token-plan-cn`）调用失败：

```
400: "The value of the enable_thinking parameter is restricted to True."
```

同 provider 下的 `deepseek-v4.1-flash`、`deepseek-v4-flash-0731` 均无此问题。

## 关键排除：不是配置里写了 thinking 字段

profile（`C:\Users\xxinj\.dsh\profiles\desktop\cordis.patch.yml`）中**没有任何 thinking 相关字段**。`enable_thinking=false` 是 DSH 内核**推断并自动发出**的。

## 根因（内核代码核实）

内核 OpenAI 兼容参数拼装处（`app.asar`，偏移约 `@116324040`）：

```js
if (compat.thinkingFormat === "zai" && model.reasoning) {
    zaiParams.thinking = options?.reasoningEffort
        ? { type: "enabled", clear_thinking: false }
        : { type: "disabled" };
    ...
}
else if (compat.thinkingFormat === "qwen" && model.reasoning) {
    params.enable_thinking = !!options?.reasoningEffort;   // 没有 effort 就发 false
    ...
}
else if (compat.thinkingFormat === "qwen-chat-template" && model.reasoning) { ... }
else if (compat.thinkingFormat === "chat-template" && model.reasoning) { ... }
```

要点：

- 每个分支的前置条件都是 `&& model.reasoning`。**这是硬门槛**，不是可选项。
- 只要 DSH 判定模型是推理模型、且本次调用未指定 `reasoningEffort`，`qwen` 分支就会发 `enable_thinking = false`。
- 百炼的 `glm-5.3` 强制要求该参数为 `True`（即不接受关闭思考）→ 400。
- `thinkingFormat` 来自**模型目录（catalog）/ provider compat**，不是 profile 的可选字段。

## catalog 状况

内置 catalog 里的 `glm-5.3` 条目全部是 **Bedrock**（`global.zai.glm-5.3`、`us.zai.glm-5.3`，`api: bedrock-converse-stream`）与 **Cloudflare AI Gateway**（`workers-ai/@cf/zai-org/glm-5.3`，`api: openai-completions`，且其 `compat` 明确 `"supportsReasoningEffort": false`）。**没有百炼上的 glm-5.3 条目**，所以 DSH 只能把它当自定义模型推断属性。

`reasoning` 是合法的模型字段：内置 catalog 里 Nova 系列即以 `"reasoning":false` 标注非推理模型（`amazon.nova-lite-v1:0`、`amazon.nova-micro-v1:0`、`amazon.nova-pro-v1:0` 及各 APAC 版本）。

## 修法（已验证）

在 profile 的 `glm-5.3` 模型条目里显式声明 `reasoning: false`：

```yaml
- id: glm-5.3
  name: GLM-5.3
  contextWindow: 1000000
  maxTokens: 65536
  reasoning: false      # ← 新增
  input:
    - text
```

原理：`model.reasoning` 为假 → 所有 `thinkingFormat` 分支都不进入 → **`enable_thinking` 根本不会被发出** → 400 无从产生。

另两个模型 `deepseek-v4-flash-0731` / `deepseek-v4.1-flash` **未改动**（`reasoning` 保持默认 true），思考开关由用户手动控制。

## 验证

改后 YAML 解析通过，18 个条目：

```
deepseek-v4-flash-0731   window=1000000 maxTokens=65536 reasoning=(default:true)
deepseek-v4.1-flash      window=1000000 maxTokens=65536 reasoning=(default:true)
glm-5.3                  window=1000000 maxTokens=65536 reasoning=false
safePointRatio: 0.85
```

压缩触发点保持 `min(850000, 1000000-65536-65536) = 850000`（85.0%），未受影响。备份：`cordis.patch.yml.bak-20261021-015450`。**须重启 DSH 生效。**

## 操作坑

- 编辑 profile 时若报 `file changed since it was read`，是 **DSH 自己重写了该文件**；重新读取最新内容再改即可，不要以为是外部冲突。
- `%TEMP%` 下的临时脚本可能被清理（`Cannot find module ...MODULE_NOT_FOUND`）；用完再跑一次前先确认文件仍在，或改用 PowerShell 一次性验证。

## 附带发现：`deepseek-v4-flash-1017` 并不存在

用户报 DSH 里 404、opencode 里可用。实测：

| 模型名 | dashscope | maas | 结论 |
|---|---|---|---|
| `deepseek-v4-flash-1017` | 404 | 404 | **不存在** |
| `deepseek-v4.1-flash-1017` | 503（服务端临时调度失败） | 超时 | 存在 |
| `deepseek-v4.1-flash` | 200 | 200 | 存在 |
| `deepseek-v4-flash-0731` | 200 | 200 | 存在 |

dashscope `/models`（共 178 个）中 DeepSeek 相关条目包含 `deepseek-v4.1-flash-1017`，**没有** `deepseek-v4-flash-1017`。opencode 配置 `C:\Users\xxinj\.config\opencode\opencode.json` 里的两个模型也正是 `deepseek-v4.1-flash` 与 `deepseek-v4.1-flash-1017`。

结论：用户输入的 `deepseek-v4-flash-1017` 少了一个 `.1`，正确名为 **`deepseek-v4.1-flash-1017`**。`503 Master scheduling failed` 是百炼服务端临时调度失败，重试通常可通，不是权限问题。

## 复用教训

- 排查"配置里明明没有某参数却被发出去"，要看**内核的推断分支**，而不是只 grep profile。
- 自定义模型（catalog 无对应条目）会被 DSH 按默认 `reasoning: true` 处理，从而踩中 provider 不容许的思考参数；显式 `reasoning: false` 是该类冲突的通用解法。
- 400 报错信息里的参数名（如 `enable_thinking`）直接指明了要查的字段名，先按名字定位内核发射点最高效。
- 模型 404 先查 `/models` 实际列表核对拼写，别先怀疑鉴权/权限。

## 相关文档

- `d920cff7-e01e-4749-b18e-89dfadb6e2f3` —— 同一 `thinkingFormat: 'qwen'` 机制的另一面（推理模型共用 max_tokens 导致正文为空、`enable_thinking:false` 作为方案 A）；该文档标注的"`llm-pi-ai` 自定义 provider 是否接受 thinking 配置尚未验证"风险，本篇以实测分支代码与配置结果补上了答案。
