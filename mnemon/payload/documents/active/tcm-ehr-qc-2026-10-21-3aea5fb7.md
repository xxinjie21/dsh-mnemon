---
id: "3aea5fb7-3feb-4bd1-8212-e6946f8b1163"
title: "tcm-ehr-qc 一致性修复落地：质控完成度判据缺陷与「改了没生效」双缓存陷阱（2026-10-21）"
description: "承接 bd486a07 排查的修复落地：qcCoverage complete 判据要求 scored==total 导致 28 条 grade IS NULL 病历使告警永不消失，改为分母只计 grade IS NOT NULL（34182→34154，complete=true）；coverage 收窄到合格病历；页面默认 importRange=null；6 个 commit 清单（c262ec8/410d8b6/7fbb442/ec8d99f/8a717a3）；Vite dev server 模块缓存与报告 SWR 缓存导致「改了不生效」的验证方法与已知工具坑（CRLF 导致 ops_edit_file 锚点失配、ops_write_file 换行丢失）。"
status: "active"
created_at: "2026-10-21T04:00:04.134Z"
updated_at: "2026-10-21T04:00:04.134Z"
content_hash: "54ba79e6b5e920de4b9d91a8fc4ff91c06bd228f4cfeea78fdb5f798bf61022c"
source_paths:
  - "frontend/src/views/StandardizationReport.vue"
  - "frontend/src/api/stats.js"
  - "java-backend/src/main/java/com/tcm/ehr/service/impl/StandardizationReportServiceImpl.java"
  - "java-backend/src/main/java/com/tcm/ehr/domain/vo/StandardizationReportVO.java"
session_ids:
  - "3bd46fb9-28ee-45ad-b5a8-5b157ec42cdc"
memory_body_ids:
  []
---

# tcm-ehr-qc 一致性修复落地（2026-10-21）

承接文档 `bd486a07-3d3c-4ecb-af6e-fc6f2f4fad5f`（只读排查 + 计划）。本轮把排查结论**落地为代码改动并实测验证**，全程 637 个测试通过。

## 一、修复 commit 清单（本地领先远端 6 个）

| commit | 问题 | 结论 |
|---|---|---|
| `c262ec8` | P0：`governed` 与 `grade` 脱钩 → 160 条病历「既不可导出也不在待清洗队列」的数据黑洞 | 已验证，不一致计数 0 |
| `410d8b6` | P1：报告归一率分母含待复核/无效病历，口径与导出口径不一致 | 分母收窄到合格病历 |
| `7fbb442` | P2：页面默认「今天导入」→ 默认范围全为 0 | 页面默认改为全库 |
| `ec8d99f` | 报告投影缺 `grade` 列，导致收窄口径后报告归零 | 补齐投影 |
| `8a717a3` | 质控完成度误判 → 永远显示「质控评分还没跑完」 | 已验证 `complete=true` |

（另一处 `importRange` 默认值改动即落在 `7fbb442` 所属的页面默认口径修复中：`frontend/src/views/StandardizationReport.vue` 的 `filters` 由 `importRange: [today, today]` 改为 `importRange: null`。）

## 二、质控完成度判据缺陷（本轮核心 bug）

**现象**：用户重跑质控后，页面仍提示「质控评分还没跑完，现在的数字不可信」，而 `qc.scored` 已达 34154/34182（99.9%）。

**判据链**（前端 `frontend/src/views/StandardizationReport.vue`）：
- `showQcGate = qcLoaded && !isEmptyReport && !qcComplete`（约 286 行）
- `qcComplete = qc.complete === true`（约 278 行）
- 文案在约 407-416 行（`title: '质控评分还没跑完，现在的数字不可信'`）。

**根因**（`java-backend/.../service/impl/StandardizationReportServiceImpl.java`，`qcCoverage(List<Row> rows)`）：
```java
c.setComplete(c.getTotal() > 0 && c.getScored() == c.getTotal());  // 要求 100%
```
`total = rows.size()` 把 **28 条 `grade IS NULL`（无可质控内容、质控永远跑不出分）** 的病历也算进分母，于是 `scored` 永远追不上 `total`，`complete` 恒为 false。

**实测对账**：`qc.total = 34182`、`qc.scored = 34154`、差值 28；`grade IS NULL = 28`、`grade IS NOT NULL = 34154` —— 28 与 28 完全吻合，确认因果。

**修复**：`qcCoverage` 的 `total` 改为**只计可质控病历**（循环内 `if (row.grade == null) continue;` 后 `c.setTotal(c.getTotal() + 1)`），即分母 = `grade IS NOT NULL`。语义从「全部病历都质控过」改为「能质控的都已质控」。
**验证**：清缓存后 `qc.total = 34154`、`qc.scored = 34154`、`qc.complete = True`，`dataset.recordCount = 28801`。

## 三、「改了没生效」的两个缓存陷阱（本轮踩了两次）

1. **Vite dev server 模块缓存**：磁盘源码已是 `importRange: null`，但 dev server 仍返回旧模块 `reactive({ department: '', dateRange: null, grade: '', importRange: [today, today] })`；`RangeFilter` 组件的「今天」按钮只在 `@click` 时触发，已排除组件内覆盖。**解决**：停掉 vite 进程（PID）、清 `.vite` 缓存、重启（`:3000`），随后 curl 该模块确认返回新代码。教训：**改前端源码后必须验证服务实际返回的模块内容，不能只看磁盘源码**。
2. **报告 SWR 缓存（TTL 180s，实际存活约 30 分钟）**：后端 Java 已改、测试通过，但接口仍返回 `total = 34182`；**清 Redis 缓存（9 个 key）后立刻变为 34154 / complete=true**。

**可复用验证纪律**：改动 → 清缓存（前端模块缓存 / 后端报告缓存）→ 用接口或 dev server 实际返回内容对账 → 才下结论。

## 四、本轮遇到的工具坑（写文件时）

- `ops_edit_file` 对该 Java 文件锚点**持续报 EDIT_CONFLICT（oldText not found）**，即使内容与预期一致、`setTotal(rows.size())` 全文仅出现 1 次、文件为 LF；同文件 git 提示 **"CRLF will be replaced by LF"**。建议：锚点失配时先用 `ops_write_file` 场景外的方式确认实际字节，或退回 Python 脚本替换。
- Python 脚本批量替换时**必须把写入放在最后**、且分步写；否则前面替换成功、后面断言失败抛异常，**文件整体未落盘**（本轮误判过一次「已改」）。
- `chr(10).join(...)` 经 `ops_write_file` → Python 读取链路会出现**换行丢失被拼成一行**，需再用单行 `ops_edit_file` 修补。

## 五、相关文件

- `frontend/src/views/StandardizationReport.vue`
- `frontend/src/api/stats.js`
- `java-backend/src/main/java/com/tcm/ehr/service/impl/StandardizationReportServiceImpl.java`
- `java-backend/src/main/java/com/tcm/ehr/domain/vo/StandardizationReportVO.java`

## 六、遗留

用户尚未确认是否推送这 6 个 commit；并需硬刷新（Ctrl+Shift+R）确认页面导入时间为空、不再显示质控告警、数字为全库口径。
