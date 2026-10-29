---
id: "5d0ffc8f-ed93-423b-a2d8-0edf42c6d5f1"
title: "tcm-ehr-qc 标准化质量报告并入首页看板：独立页与 CSV 导出下线（2026-10-25，commit 71041f0）"
description: "批次 28 收尾重构：标准化质量报告从独立路由页改为首页 Dashboard 的两个 StatCard + 默认收起的 PanelCard（复用 StandardizationCharts，接口并行拉取单独 catch 成 null）；同步删除 StandardizationReport.vue/StandardizationDetails.vue、三套菜单项、MainLayout 图标映射、AuthServiceImpl pathOf 映射、CSV 导出一整套（StatsController 端点 + IStandardizationReportService/Impl 两个 reportCsv 重载 + 实现 + 测试）；删测试数据导致 AuthServiceTest 菜单数断言 11→10、10→9 失败，632 测试通过、vite build 8.78s；含「字符串匹配删代码删错方法边界、git 恢复后改大括号深度计数」的可复用事故教训。"
status: "active"
created_at: "2026-10-24T16:59:39.723Z"
updated_at: "2026-10-24T16:59:39.723Z"
content_hash: "087da9871fb8863bfc64457f6a2efa4b3c48a92bd381532e1d457d0edc9db07c"
source_paths:
  - "frontend/src/views/Dashboard.vue"
  - "frontend/src/components/StandardizationCharts.vue"
  - "frontend/src/layouts/MainLayout.vue"
  - "frontend/src/api/stats.js"
  - "java-backend/src/main/java/com/tcm/ehr/controller/StatsController.java"
  - "java-backend/src/main/java/com/tcm/ehr/service/IStandardizationReportService.java"
  - "java-backend/src/main/java/com/tcm/ehr/service/impl/StandardizationReportServiceImpl.java"
  - "java-backend/src/test/java/com/tcm/ehr/service/AuthServiceTest.java"
session_ids:
  - "8bd50f77-4e9b-4f04-a835-5208850a492b"
memory_body_ids:
  []
---

# 标准化质量报告并入首页看板：独立页与 CSV 导出一并下线

批次 28 收尾重构，提交 `71041f0` `refactor(report): 标准化质量报告并入首页看板，撤掉独立页与 CSV 导出`（13 files, +87 / −1178）。后端统计实现与 VO 一行未动，前端构建与 632 个后端测试通过。

## 一、改了什么

### 首页（Dashboard.vue）新增
- **StatCard 4 术语归一率** —— note 明确「分母是实体数，与上方合格率分母不同」（两个百分比同屏但口径不同，必须写清，否则必然被误读）。
- **StatCard 5 覆盖最低字段** —— 只在 `total > 0` 的字段里比；没抽到实体的字段不参与排名（避免 `治法 2/5=40%` 这种样本极小的字段抢占「最差」）。
- **PanelCard「术语标准化覆盖」** —— 复用原报告页的 `StandardizationCharts` 组件，**默认收起**，想看的人点一下即有。
- **数据加载** —— 报告接口与首页原有三块**并行拉取**，单独失败时 `catch` 成 `null`，不整页失败。

### 删除（总计 −1178 行）
| 项 | 说明 |
|---|---|
| `frontend/src/views/StandardizationReport.vue` | 921 行整页删除 |
| `frontend/src/components/StandardizationDetails.vue` | 296 行明细组件删除 |
| 路由 + 3 套菜单项 | 含 `MainLayout.vue` 图标映射表条目、`AuthServiceImpl` 的 `pathOf` 菜单映射 |
| **CSV 导出一整套** | 前端 API + `StatsController` 的 `/standardization-report/export` 端点 + `IStandardizationReportService`/`Impl` 两个 `reportCsv` 重载 + 实现（约 70 行）+ `reportCsvHasBomHeaderAndDisclaimer` 测试 |
| `DictionaryImport.vue` 的「查看质量报告」链接 | 改为指向首页（导入完词表后的出口不能断） |

### 保留
`StandardizationCharts.vue`（首页复用）、`GET /stats/standardization-report` 数据接口（图表数据源，**只撤页面不撤接口**）、后端 `StandardizationReportServiceImpl` + VO（一行未动）。

## 二、连带修正（删数据触发删断言）

删除菜单项后 `AuthServiceTest` 3 个用例失败 —— 断言的菜单数量/内容还是旧的：
- `adminLoginOkAndTokenValid`：`menus.size()` 11→10，删掉 `contains("标准化质量报告")` 断言；
- `adminOwnerAlsoGetsMyGroupMenu`：11→10；
- `adminWithoutGroupGetsNoGroupMenu`：10→9。
- 另有两处**说明性注释**（解释「为什么少了一项」）刻意保留：`// （批次 28 起「标准化质量报告」并入首页，不再单列菜单）`。**断言要跟着删，解释性注释要留着**，否则下次再看菜单数又会以为是漏改。

## 三、事故与可复用教训（本轮第 3 次同类错误）

**删除 `reportCsv` 实现时，结束定位匹配到了方法内部的一个 `}`（内层 for 循环的结束）**，导致方法签名被删、函数体留下 70 行残骸，文件语法破坏。前两次同类错误分别是 CRLF/LF 行尾不匹配导致批量替换 0 命中、`git add -A` 范围过大。

**根本原因：一直在用「字符串/行号匹配」做代码删除，而不是结构化方式。**

修正后的操作守则：
1. **删代码前先用括号配对确认边界** —— 大括号深度计数或 brace-matching，不要信行号也不要信单行字符串；
2. **确认文件行尾**（本仓库 Java 文件多为 CRLF），批量替换前先探测，LF 模式匹配 CRLF 文件会静默 0 命中；
3. 删坏了**立刻 `git checkout` 恢复整个文件**，不要在残骸上继续打补丁；
4. 删除后做**衔接检查**（看前后 10 行是否自然接入下一个方法），再编译。

## 四、验证结果

| 项 | 结果 |
|---|---|
| `mvn clean test` | ✅ 632 通过（原 633，减 1 个 reportCsv 测试） |
| `vite build` | ✅ 8.78s |
| `standardization-report` 路由/页面/导出残留 | ✅ 0（仅剩 `/stats/standardization-report` 数据接口） |
| `reportCsv` / `exportStandardizationReport` 全库残留 | ✅ 0 |
| `StandardizationReport` 92 处残留 | 全部是**要保留的类型名/文件名**（`StandardizationReportServiceImpl`、`StandardizationReportVO` 等），非页面引用 |

## 五、与既有口径文档的关系（必读）

本重构只改前端载体，**不改统计口径**。以下既有结论继续有效，改首页卡片文案前先读：

- `bd486a07-3d3c-4ecb-af6e-fc6f2f4fad5f`：页面默认 `importStart=importEnd=今天` 导致空数据的陷阱；`governed=1` 与 `grade` 脱钩的 160 条数据黑洞；`recordCount=rows.size()` 分母含未质控/待复核/无效病历的语义问题。
- `2c4d269c-9de4-4a61-b5c0-11e73f144450`：coverage 归一率（分母=全部已抽取实体，症状 39.8%）与 normalizable（分母=词表内实体，恒 ~100%）两个同名不同义指标；`StandardizationCharts` 「条长=归一率」的视觉误读与候选改法（柱尾标值 / 90% 参考线 / 并列未归一绝对条数）。
- `fbdfeb24-623f-4d20-9ae2-3cab0be26eb2`：未归一 84105 条按责任方六桶拆解（dictionaryGap / inquiryFragment / processDesc / misrouted / physicalSign / fragment，守恒校验）。

首页 StatCard 4 的「术语归一率」取的正是 coverage 口径（实体分母），**不是** normalizable 的算法口径 —— 这正是 `2c4d269c-9de4-4a61-b5c0-11e73f144450` 警告过的误读点，卡片 note 已显式区分分母。
