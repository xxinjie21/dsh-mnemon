---
id: "bf2bcf5d-39e1-4c5d-b005-0d8452bb1149"
title: "dsh-mnemon profile 配置最小化：cordis.patch.yml 中 mnemon 块的必需键与默认值对照"
description: "DSH desktop profile 中 dsh-mnemon 插件配置收敛为最小必需键（storageScope/dataDir + 两个策略开关）的证据、保留理由与默认值来源对照表。"
status: "active"
created_at: "2026-10-19T16:12:53.031Z"
updated_at: "2026-10-19T16:12:53.031Z"
content_hash: "6bb4c335f3c228cc5ce5733109fe563d01b5fc000be736c2bbda9498ea743f6c"
source_paths:
  []
session_ids:
  - "8e1c7169-d4e9-42ca-a135-10d60a4af1d4"
memory_body_ids:
  []
---

# dsh-mnemon profile 配置最小化（cordis.patch.yml）

## 结论

DSH desktop profile 里 `dsh-mnemon` 的配置块，实际只有 2 个键是**非默认、必需**的，其余全部可以删除并回落到 schema 默认值：

```yaml
- id: mnemon
  name: dsh-mnemon
  config:
    storageScope: custom
    dataDir: D:\deepseekmnemon
```

同文件中另有三行是**真开关**，必须显式保留（bundle 内这两个增强默认关闭）：

```yaml
- id: mnemon-strategy-light-context
  disabled: false
- id: mnemon-strategy-auto-capture
  disabled: false
- id: michengai-simplify
  disabled: true
```

收敛结果：mnemon 配置块 33 行 → 2 行，整文件 131 行 → 101 行；非 mnemon 行逐字节未变。

## 必需键的理由与来源

| 键 | 为什么必须保留 | 来源 |
| --- | --- | --- |
| `storageScope: custom` | 默认是 `"global"` | `dsh-mnemon/lib/index.js:708`（`?? "global"` 兜底） |
| `dataDir: D:\deepseekmnemon` | `storageScope: custom` 时**必填**，缺失会让 `resolveConfig` 直接抛错；且实际数据（documents/runtime/state 三个区）已在此目录，删除等于换存储位置、旧数据不会迁移 | `dsh-mnemon/lib/index.js:720` |

配置文件路径为 `%USERPROFILE%\.dsh\profiles\desktop\cordis.patch.yml`，其依赖位于同目录 `node_modules/` 下。

`include:` 前缀不是野键：它是 DSH 组合层给 group 内嵌套条目的 id 前缀（见 app.asar 中 `entryId.replace(/^include:/, "")`）。但示例中的两条 `include:*` 条目带**空 config 且无 `enabled`**，不贡献任何行为，属纯噪音。

## 删除项的默认值对照（机器逐条比对，全部等同默认）

默认值权威来源：zod schema `Config` 与 `resolveConfig` 兜底在 `dsh-mnemon/lib/index.js:510-578` 与 `:700-751`；recall 质量默认在 `dsh-mnemon-source-memory-spaces/lib/index.js:3429-3451`。

| 删除的键 | 原值 | 等同的默认 |
| --- | --- | --- |
| `remoteAccess` | read-only | `.default("read-only")` `:560` |
| `runtimeUserScope` | storage | `.default("storage")` `:517` |
| `timeoutMs` | 10000 | `DEFAULT_TIMEOUT_MS = 1e4` `:418` |
| `defaultRecallLimit` | 10 | `?? 10` `:729` |
| `embedding` | enabled:false, http://localhost:11434, nomic-embed-text | schema 默认 `:534-540` |
| `recallQuality` | strict-v1 / 0.25 / 0.6 / 3 / 4 / 2 | `resolveRecallQuality(void 0)`，六个值逐一相同 |
| `routingGuidance`,`tabEnabled`,`writeEnabled`,`lifecycleEnabled` | true | `.default(true)` `:552/558/559/561` |
| `recallMode`,`writebackMode` | guided | `.default("guided")` `:562/563` |
| `idleReviewMs` | 30000 | `DEFAULT_IDLE_REVIEW_MS = 3e4` `:412` |
| `memoryView.strategyTypeId` | default-three-tier | 兜底 `?? "default-three-tier"` `:15028` |
| `memoryView.entries.include:*` | `{config: {}}` | 空 config + 无 `enabled` ⇒ 不生效（`view()` `:15828-15834` 仅在 `typeof item.enabled === "boolean"` 时生效） |

## 需要人工确认的取舍

`syncEnabled: true` 被一并删除（回落默认 `false`）。证据表明在未配置 git 时该开关是惰性的：

- 无 `repoUrl` 时自动备份为空转：`if (sync.settings().read().repoUrl === void 0) { ... }`（`:865-867`）
- 定时器根本不会启动：`autoBackupMinutes` 默认 0，`if (minutes <= 0) return`（`:843`）

它实际只影响设置页是否渲染 Git 同步区块。若计划启用 Git 同步，需把它加回配置。

## 操作与验证注意事项

- 配置在 DSH 启动时读取，**修改后需要重启**才生效。
- `cordis.patch.yml` 由 App/插件管理器持有并可能回写：编辑过程中出现过 `file changed since it was read`；重启后若默认值行被 UI 写回属正常现象，不影响正确性。
- 变更前备份：`cordis.patch.yml.bak-20261020-001035`（原 sha256 `A730F390…`，变更后 `FAFD00FF…`）。
- 验证方式：YAML 解析确认顶层为 `array(14)`、mnemon 行 config 键恰为 `storageScope, dataDir`；非 mnemon 行逐字节未变。
