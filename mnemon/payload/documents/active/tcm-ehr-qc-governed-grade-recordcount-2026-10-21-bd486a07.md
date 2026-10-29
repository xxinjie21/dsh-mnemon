---
id: "bd486a07-3d3c-4ecb-af6e-fc6f2f4fad5f"
title: "tcm-ehr-qc 标准化质量报告一致性排查：governed/grade 脱钩、recordCount 口径与默认导入时间陷阱（2026-10-21）"
description: "报告页与助手数字不一致的根因（页面默认 importStart=importEnd=今天，全库 vs 空数据两套口径实测值）；governed=1 与 grade 脱钩致 160 条待复核病历既不可导出也不在待清洗队列的数据黑洞；recordCount=rows.size() 含未质控/待复核/无效病历的分母语义问题；StatsCache(SWR, TTl 180s) 缓存一致性经验证无问题；P0/P1/P2 修复计划与排查方法经验。"
status: "active"
created_at: "2026-10-21T03:06:42.843Z"
updated_at: "2026-10-21T03:06:42.843Z"
content_hash: "43deb7d177ff2f3d6d7be90bc156f537148694259b4bbbd79cb644a9f747d021"
source_paths:
  - "frontend/src/api/stats.js"
  - "frontend/src/views/StandardizationReport.vue"
  - "java-backend/src/main/java/com/tcm/ehr/service/StandardizationReportService.java"
session_ids:
  - "f9cd3cfc-2f9a-42d0-a524-3504019e05f6"
memory_body_ids:
  []
---

# 标准化质量报告一致性排查：governed/grade 脱钩、recordCount 口径与页面默认时间陷阱（2026-10-21）

## 背景与触发
用户问：为什么助手给的「归一率」与标准化质量报告页面显示的数据不一样。排查后确认页面与助手调用的是**同一个接口** `/api/stats/standardization-report`，差异不在接口而在**分母口径**。随后用户要求系统性排查该页面的缓存、数据库一致性、数据同步性与实时性，并先出计划再改。
本轮为**只读排查 + 计划**，未做任何代码改动。

## 结论一（根因）：页面默认口径是「今天导入」，不是全库
- 前端注释即判据：`frontend/src/api/stats.js` 第 21-25 行 —— 「页面默认 `importStart=importEnd=今天`（不分科室）；甲类（词典质量）字典全域、不随范围收窄，乙类随 department/时间收窄」。
- 实测 A（模拟页面默认，import=今天）：`dataset.recordCount = 0`，`coverage[]` 各项 `total=0, normalized=0`，`byMonth=[]`。
- 实测 B（不带参数，全库）：`recordCount = 34182`；证候 34724/38732 = 90.8%、症状 106574/195403 = 54.5%、中药 436873/439045 = 99.5%、疾病 89.7%、舌象 93.7%、脉象 96.1%、病因 99.9%、方剂 805/936 = 86.0%、治法 2/5 = 40.0%。
- 数据库导入时间分布：仅 `2026-10-15: 500`、`2026-10-14: 33682`；当天（2026-10-21）**无导入**，故页面默认口径全为空。
- 两个数字**都是对的**，只是口径不同。用户自行解析/归一后看到的页面空数据并非解析失败。

## 结论二：清洗标记 `governed=1` 与 `grade` 脱钩 → 160 条病历成数据黑洞
交叉分布（34182 条）：

| grade | governed=0 | governed=1 |
|---|---|---|
| 合格 | 0 | 28793 |
| 待复核 | 1678 | **160** ⚠️ |
| 无效 | 3523 | 0 |
| NULL | 28 | 0 |

- 样例：`('00a8587a-…', '待复核', 1, 'reviewing')` 等，`status=reviewing`。
- 根因：病历**曾判合格并被清洗**（governed=1），**后经质控改判为待复核**，但 `governed` 标记**未撤销**。
- 后果：导出要求 `grade=合格 AND governed=1`，待清洗队列要求 `grade=合格 AND governed=0` —— 这 160 条**两边都不进**，既不可导出也不在待清洗队列，用户看不到也处理不了。
- 量级叠加：`governed=1` 总 28953 > `合格` 28793，正是这 160 条的差。

## 结论三：报告 `recordCount` 语义模糊（分母含不该参与的病历）
- 报告 `dataset.recordCount = 34182`（全部病历），数据库「有结构化数据」仅 **34087**，差 95 条。
- 源码位置：`java-backend/src/main/java/com/tcm/ehr/service/StandardizationReportService.java`
  - 第 912 行 `s.setRecordCount(rows.size())`（是查询行数，不是全表数）；
  - 第 643-653 行 `rowsInDomain` 用 `RecordFilter.buildForAggregate(...)`，**不带 grade / structured 过滤**，只投影 8 列（id, visit_time, score, update_time, chief_complaint, self_report, structured_data, qc_results）；注释说明刻意不 ORDER BY（避免 filesort），且数据域与「看全部」走 `ReportContext` 快照（SWR 后台线程读不到请求上下文）。
- 后果：未质控 28 条、待复核/无效 5361 条也进入分母 —— 归一率的**分母口径因此被稀释**，页面「病历总数」容易被读作「已分析的数据集规模」。
- 另注：`recordCount` 与 SQL 直算「有结构化数据」不同源，今后对账须先确认「行在不在域内 + 有没有 grade 过滤」。

## 结论四：缓存一致性经验证**无问题**
- 报告缓存：`StatsCache#getOrLoadSwr("report", f, dictVersion, REPORT_TTL)`，`REPORT_TTL = 180s`（带 ±10s 抖动，避免同秒集中过期）；SWR 硬过期倍率使 dataKey 实际存活约 30 分钟，过期后**立即返回旧值 + 后台单飞重算**。
- salt 用 `termStore.effectiveDictVersion(ctx.orgId())`；失效走 StatsCache 全局版本段，任一写入（导入/清洗/复核/批次）即作废。
- Redis 实测：`tcm:ehr:cache:stats:report:v0:*` 共 5 个 key，TTL 正常。
- 对账实测：同一口径连查两次结果一致（第二次更快=命中）；清缓存前后结果一致 → **无陈旧值**。

## 排查方法经验（可复用）
- `frontend/src/api/stats.js` 这类**前端注释就是口径判据**：接口默认值往往写在注释里，先读注释再猜后端。
- 对账顺序：接口不带参数 vs 模拟页面默认参数 → 数据库 SQL 直算 → 交叉分布表 → 清缓存对比；不要先改代码。
- Windows 环境下既有的坑：`bash -c` 里写含 `()` 和引号的 SQL 会语法错误，Python 拼 SQL 字符串也会炸；正确做法是**写成脚本文件执行 + SQL 参数化**。
- 报告查询 `rowsInDomain` 只投影 8 列是有意的（全列 21 TEXT + 2 JSON、4 万条约 45MB），改动分母口径时不要顺手放开投影。

## 修复计划（已提出，等待用户拍板，未实施）
| 优先级 | 问题 | 修法 | 影响 |
|---|---|---|---|
| P0 | `governed` 与 `grade` 脱钩（160 条黑洞）| 质控改判时撤销 governed；或导出/待清洗条件统一以 `grade=合格` 为准 | 数据完整性 |
| P1 | 报告 `recordCount` 语义 | 拆成「总病历」与「已质控合格」两个指标，或把报告分母收窄到合格 | 口径准确性 |
| P2 | 页面默认口径 | 默认改为「不限时间」，或空数据时明确提示「今天无导入数据，请调整导入时间范围」 | 体验 |

- P1 尚需用户决策：分母算全部（反映「系统处理了多少数据」）还是只算合格（反映「标准化数据集的质量」，助手倾向后者）。
- P2 备选方案：A 默认「不限时间」（助手建议）；B 保持默认但空数据时给明确提示。

## 关联文档
- `2c4d269c-9de4-4a61-b5c0-11e73f144450`：同一报告页 coverage / normalizable 两个「归一率」口径辨析与图表视觉误读排查（2026-10-20）。其「先对账后端口径与 SQL 再改前端」的经验与本轮同源，本文档补充的是**范围口径（导入时间/grade）**与**缓存实时性**两个新维度。
- `e2138429-3421-48b9-b77f-ccc47c387b6a`：全库重抽闭环与九字段归一率基线（本轮全库实测与之对齐）。
