---
id: "7bec5266-1740-4f7d-a252-7ad7a78094e4"
title: ".agents/skills 与 aegis 插件冲突治理：同名 skill 取舍与 trigger 隔离（结论与证据）"
description: "用户级 .agents/skills 目录由 34 个收敛到 4 个、消除与 aegis 插件 23 个 skill 的同名冲突、隔离 caveman 与 communicating-concisely 重复触发词的判断依据、加载路径事实与操作记录。"
status: "active"
created_at: "2026-10-19T16:45:49.007Z"
updated_at: "2026-10-19T16:45:49.007Z"
content_hash: "05c3ff5723e2f1660faaf54b680c85cb780b5dda51ac0ea0107a7606d32bb908"
source_paths:
  - ".agents/skills/caveman/SKILL.md"
session_ids:
  - "97451fab-e1f8-4efa-920e-c28499abaf9c"
memory_body_ids:
  []
---

# .agents/skills 与 aegis 插件冲突治理

结论记录：用户级 skill 目录 `C:\Users\xxinj\.agents\skills` 与 DSH 插件包 `aegis`（v2.13.1）自带的 skill 树如何共存，以及最终取舍与证据。

## 环境与加载路径事实（本次核实）

- 公共（用户级）skill 根：`C:\Users\xxinj\.agents\skills`，被 `.claude\skills` 以 Junction 链接复用（两边目录数一致）。
- aegis 的 23 个 skill 来自 profile `node_modules` 内的插件包，通过其 `extensions/dsh/cordis.patch.yml` 注入一个整体 entry：
  ```yaml
  - insert:
      - id: aegis-method-pack
        name: 'aegis/extensions/dsh/index.js'
  ```
  → 没有逐 skill 开关；plugin 直接读取整个 `skills/` 树，无法单独禁用其中某个 skill。
- aegis 的 skill 不在 `.agents` 下，skills-manager 的 `sources`（21 个外部 agent 根 + dsh）不覆盖它 → **无法用 `disabledSkills` 或来源开关关掉 aegis 侧**。
- 因此消除同名冲突在架构上只有一条路：删除 `.agents\skills` 中的同名项，保留 aegis 版（这也符合 aegis 自身“bundle 激活时不暴露影子根副本”的约定）。

## 第一轮：目录收敛（34 → 10 → 后为 4）

1. 先按白名单把 `.agents\skills` 恢复/整理为 34 个、621.4 KB，并创建备份 `C:\Users\xxinj\.agents\skills-backup-20261020-003512`。
2. 按白名单删除 23 个，剩余 10 个（250.1 KB），与白名单精确匹配。
3. 用户要求删备份。删除前确认：只有一个备份目录、`LinkType=` 为空（普通目录非链接）、正式目录 10 个不受影响 → 删除成功；此后 `.agents` 下只剩 `skills`。
   - 后果：删掉的 23 个 skill 无本地副本；需要时只能从上游重新获取（superpowers、caveman、ponytail 各有公开仓库或 npm 包）。

## 第二轮：与 aegis 的冲突判定

10 个公共 skill 中 6 个与 aegis 23 个同名：

| 同名 skill | 公共版体积 | aegis 版体积 | 结果 |
|---|---|---|---|
| brainstorming | 10047 B | 25185 B | 删公共版 |
| systematic-debugging | 9465 B | 11381 B | 删公共版 |
| test-driven-development | 9015 B | 18169 B | 删公共版 |
| using-git-worktrees | 6813 B | 7779 B | 删公共版 |
| verification-before-completion | 3646 B | 8762 B | 删公共版 |
| writing-plans | 6907 B | 13528 B | 删公共版 |

**取舍理由（重要，勿用“体积更大=更好”复述）**：这 6 对不是“通用版 vs 增强版”，而是“无条件强制版 vs 带门控的 Aegis 版”，语义和触发门槛都不同：

- 公共 `test-driven-development`：无条件 “write test first, watch it fail”；aegis 版开头即 *“Exit immediately unless the user explicitly asked for TDD”*，且仅认 `TDD Route: strict`。
- 公共 `verification-before-completion`：无条件“证据优先”；aegis 版带 `<EXPLICIT-MODE-GATE>`，非显式调用直接回 fast path。
- 公共 `brainstorming`：“MUST use before any creative work”；aegis 版仅对 ambiguous / high-complexity 生效，小改动走 fast path。
- 公共 `writing-plans`：多步任务即写计划；aegis 版要求已有批准的 spec。
- 公共 `systematic-debugging`：root-cause 原则（“symptom fixes are failure”）；aegis 版是 L1 symptom → … → L7 spec gap 分层诊断。
- 公共 `using-git-worktrees`：需要隔离即建 worktree；aegis 版加了“何时才真需要 worktree”的门槛。

关键判据：用户当时 `TDD mode: off`，保留公共版 TDD skill 等于绕过 aegis 门控。同名双份的真正危害是**加载歧义**，而非质量高下。

## 第三轮：caveman vs communicating-concisely（不同名、同触发词）

- 公共 `caveman` 与 aegis `communicating-concisely` 触发词完全重叠：`caveman mode` / `talk like caveman` / `use caveman` / `less tokens` / `be brief`。
- 内容对比结论：**caveman 严格更好**——6 档强度（lite/full/ultra/wenyan-*）、`Preserve user's dominant language exactly`、`Never drop not/never/no/only/except`（防语义反转）、明确禁止编造缩写（cfg/impl/req/res/fn 在 tokenizer 下与全词 token 数相同）。
- aegis 的 `communicating-concisely` 反而**鼓励**滥用缩写和 `X → Y` 箭头，还附了一段与表达无关的“降低内容审核风险”说法。
- 但因 aegis 侧不可禁、不可改，只能二选一。用户选择：**保留 caveman，改写其触发词避免竞争**。

### 采用的改法（`C:\Users\xxinj\.agents\skills\caveman\SKILL.md` frontmatter description）

- 仅由 `/caveman`（可带档位）、显式点名（“caveman skill”、“用 caveman”）、或切换 caveman 强度的请求触发。
- 通用简洁类说法明确让给 `communicating-concisely`（defer，不抢占）。
- 第一版曾在 description 中写出原触发词原文作为“反例说明”，后判定有被匹配风险（description 是模型选 skill 的主要依据，出现即可能命中），改为**不含原文**的表述。最终：89 行、6398 B，frontmatter 分隔符合法，`name=caveman`，旧触发词残留检查为 none。

## 最终状态

`.agents\skills` 4 个 / 与 aegis 同名交集为 0：

```
agent-browser    3415 B   浏览器自动化（aegis 无对应）
caveman          6398 B   压缩表达（触发词已隔离）
ponytail         6757 B   YAGNI / 防过度工程
prd-generator   15345 B   PRD 生成
```

`.claude\skills` Junction 同步显示 4 个。

## 可复用的操作经验

- 判断 skill 与插件冲突时，先查 **加载路径与可控开关**（插件 patch 是整体 entry 还是逐项、是否在 skills-manager 的 sources 内），再谈内容取舍。
- skill 的 `description` 会被模型用于选择；想“排除”某组触发词，不要在 description 里引用这些词原文，改用描述性表述。
- 删除任何备份/目录前：列 `.agents` 下全部备份目录、检查 `LinkType`（排除 Junction/符号链接）、核对正式目录完好性，再删。
- 含中文引号的 PowerShell 单行命令易被解析错乱 → 写成 `.ps1` 脚本文件再执行。
