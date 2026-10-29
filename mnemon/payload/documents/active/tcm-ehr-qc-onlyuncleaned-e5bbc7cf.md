---
id: "e5bbc7cf-31f0-435d-a73f-99c432334600"
title: "tcm-ehr-qc 清洗支持「只清洗未清洗的」（onlyUncleaned）实现与端点实测"
description: "Governance 清洗新增 governed=0 范围过滤：FiltersDTO.onlyUncleaned → RecordFilter.andOnlyUncleaned → qualifiedScope 透传 → 前端独立勾选框；实测未清洗 18 条 vs 全范围 28793 条，637 测试通过，commit 0961278；含错加 SearchDTO 重载的编译期坑与重抽后归一状态复核口径（全库 88.3% vs 前 3000 条抽样 74.5%）。"
status: "active"
created_at: "2026-10-21T02:11:58.452Z"
updated_at: "2026-10-21T02:11:58.452Z"
content_hash: "51eb9778744fb37640fe9c74e81bf79c6508e649d66f00558ecee6c51e6836ab"
source_paths:
  - "java-backend/src/main/java/com/tcm/ehr/domain/dto/FiltersDTO.java"
  - "java-backend/src/main/java/com/tcm/ehr/service/impl/GovernanceServiceImpl.java"
  - "java-backend/src/main/java/com/tcm/ehr/mapper/RecordMapper.java"
  - "java-backend/src/main/resources/db/schema.sql"
  - "frontend/src/views/Governance.vue"
session_ids:
  - "aafa889f-c2f7-4a61-83fb-1087ec009ee7"
memory_body_ids:
  []
---

# tcm-ehr-qc 清洗支持「只清洗未清洗的」（onlyUncleaned）

## 需求来源

用户要求：**清洗与导出要能单独对「没清洗过」的病历执行清洗**，而不是每次都在全库范围内重跑。

## 现状诊断（改动前）

| 项 | 现状 |
|---|---|
| 数据库 | `records.governed TINYINT NOT NULL DEFAULT 0`（已清洗标记，"清洗 + 术语归一完成后置 1"），索引 `idx_records_org_grade_gov (org_id, grade, governed)` |
| 统计 | `RecordMapper` 已按 `grade='合格' AND governed IN (0,1)` 聚合出 pendingGovern / governedCount |
| 导出 | 已按 `governed = 1` 过滤（清洗未完成的不算标准数据集） |
| **清洗范围** | `GovernanceServiceImpl.qualifiedScope(FiltersDTO)` 只设 `grade="合格"`，**没有任何"只未清洗"条件** → 每次全范围重跑 |

`FiltersDTO` 原有字段：department、dateRange、importRange、grade、ids、onlyUnparsed。其中 `onlyUnparsed`（`structured_data IS NULL`）是"只处理未处理的"这一类需求的既有模板。

## 实现（4 处改动，commit `0961278`，4 files changed, 41 insertions(+), 1 deletion(-)）

1. **`java-backend/src/main/java/com/tcm/ehr/domain/dto/FiltersDTO.java`**：新增 `Boolean onlyUncleaned`，语义为"仅处理尚未清洗的病历（`governed = 0`）"，缺省/false 时不加条件。
2. **`java-backend/src/main/java/com/tcm/ehr/.../RecordFilter.java`**：新增私有方法 `andOnlyUncleaned(QueryWrapper<Record>, Boolean)`，仿 `andOnlyUnparsed` 写法，`Boolean.TRUE.equals(...)` 时 `wrapper.eq("governed", 0)`；在 `build(String orgId, FiltersDTO filterDto)` 重载中调用。
3. **`GovernanceServiceImpl.qualifiedScope`**：把 `filters.getOnlyUncleaned()` 复制进 scope（该方法刻意复制一份 FiltersDTO 而非就地改分级，以免操作日志范围与实际执行不一致）。
4. **`frontend/src/views/Governance.vue`**：清洗操作区新增第二个选项行（与「执行术语归一」开关同层级、独立一行）的勾选框「只清洗未清洗的」+ 传参 + 提示文案；并同步处理 `restoreRunningClean` 刷新恢复时的开关状态回填。

### 实现中踩的坑（值得复用）

初次替换时误把 `andOnlyUncleaned(wrapper, dto.getOnlyUncleaned())` 加进了 **`build(String orgId, SearchDTO dto)`** 重载 —— `SearchDTO` 没有该字段，**会在编译期失败**。清洗走的是 `FiltersDTO` 重载，不需要改 `SearchDTO`。已撤销，最终调用点只有 1 处。

**通用教训**：`RecordFilter` 有两个同名 `build` 重载（`SearchDTO` / `FiltersDTO`），批量插入新条件时必须先确认所在方法的参数类型 —— 仅凭就近的 `andOnlyUnparsed(...)` 行定位会被重载迷惑。

## 验证

- `mvn clean test`：**637 tests，0 failures，0 errors，BUILD SUCCESS**。
- 前端 `vite build`：通过（built in 7.86s）。
- **端到端实测（后端已重启）**，当时合格 28801 / 待清洗 18 / 已清洗 28783：
  - `onlyUncleaned=true` → 任务 `2c74f8dd-9ab2-45b2-94af-62ab592b6a13` 终态 `COMPLETED done=18/18`，**约 8 秒**；
  - `onlyUncleaned=false`（默认）→ 全范围 total=28793，处理中观测 done 递增到 8600+，**原行为未被破坏**。
  - 范围收窄后 `Task.total` 随之变小，进度百分比因此正确（totalSink 依赖该 total）。

## 相关重抽与归一状态复核（同日）

- 用户自行完成了重抽。复核报告覆盖率：中药 99.5%、病因 99.9%、脉象 96.1%、舌象 93.7%、证候 90.8%、疾病 89.7%、方剂 86.0%、症状 54.5%、治法 40.0%。
- 全库综合归一率 **846966 / 959528 = 88.3%**（较此前 87.8% 提升 0.5pp）；证候实体数 98796 → **103318（+4522）**，确认重抽生效。
- **口径教训**：直接抽样前 3000 条得到的 74.5% 与全库值 88.3% 相差 13.8pp，是**抽样偏差**（前段数据恰含未归一病历）。判断全库归一状态应使用 reports 覆盖率聚合或全库统计，不要用头部抽样。
- 抽取层不产出归一结果，因此**重抽会重置 `structured_data` 的 normLevel，重抽后需要再跑「仅归一」**；若重抽被中途取消，已处理的那批病历会对不上归一状态。

## 待办 / 后续

- commit `0961278` 当时**仅本地，未推送**（领先远端 1 个 commit）。
- `onlyUncleaned` 的典型用法：全库清洗完成后，新导入的病历用该开关只清洗增量。
