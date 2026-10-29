---
id: "5440bc2e-50ab-4420-906c-bd897d9dfdc5"
title: "tcm-ehr-qc 首次「仅归一」批跑实测：病因归一率 0%→62.5%，剩余缺口为真实词表缺口"
description: "NORMALIZE_ONLY 接口与参数、全库任务 34182/34154/93 条永久失败终态、9 字段归一率前后对比（cause 0%→62.5%）、剩余 11576 条未归一病因清单与补词判断（饮食需甄别）、看报告前须清 SWR 缓存。"
status: "active"
created_at: "2026-10-20T02:57:06.200Z"
updated_at: "2026-10-20T02:57:06.200Z"
content_hash: "17f2fdc55d063959192de17788797237fd7fb7fb2dad64b03a4fbaff547467eb"
source_paths:
  - "java-backend/src/main/java/com/tcm/ehr/controller/NlpController.java"
  - "java-backend/src/main/java/com/tcm/ehr/domain/po/NlpTask.java"
session_ids:
  - "427a248a-8ee7-4710-96a1-9926f2e5e7d7"
memory_body_ids:
  []
---

# tcm-ehr-qc 首次「仅归一」批跑实测结果（NORMALIZE_ONLY）：病因归一率 0% → 62.5%

## 背景
依据文档 `01b35093-502c-4d8f-8334-a88c8dacc23b` 判定：病因（cause）0% 归一的根因是归一范围（scope）不含 cause，而非词表不全；修复 scope 后跑一次「仅归一」即可让存量 `structured_data` 重算，无需全库重抽。本次检查点即为该判定第一次端到端执行与验证，结论成立。

## 任务入口与参数
- 接口：`POST /api/nlp/extract/batch/normalize-only`（`NlpController`，约 197-207 行）
- 强制模式：`NlpTask.MODE_NORMALIZE_ONLY = "NORMALIZE_ONLY"`；历史任务 mode 为 null 时按 `MODE_EXTRACT` 处理（`NlpTask#modeOrDefault`）
- 入参 `NlpBatchDTO` 可空 = 全库范围；`limit=0` 表示不限定条数
- 与批量解析共用同一套任务表、进度与失败清单，仅任务模式不同
- 进度查询：`GET /api/nlp/extract/batch/{id}`；另有 `/{id}/cancel`、`/{id}/retry-failed`、列表 `GET /extract/batch`
- 直接调用后端 API 提交，不要在前端点按钮（本仓库工作流习惯）

## 本次任务结果
- 任务 ID `f5eb4f15-1637-45bc-8388-34d6a6cccf95`，范围全库
- total 34182 / done 34154 / success 34061 / **failed 93**
- 93 条失败原因全部是「该病历尚无结构化数据，请先批量解析」→ **永久失败**，符合预期（这些病历要先跑批量解析）
- 与先前基线数字完全一致（34182 / 34154 / 93）

## 归一率前后对比（同一分母：34154 完成）
| 字段 | 归一率 | 说明 |
|---|---|---|
| causeList 病因 | **0% → 62.5%**（19286 / 30862） | 本次最大收益，验证 scope 缺 cause 是主因 |
| diseases | 82.7% | 持平（82.5→82.7） |
| symptoms | 39.8% | 略升（36.5→39.8，+3.3pp），仍是最大短板 |
| tongueList | 78.4% | 75.9→78.4 |
| pulseList | 91.7% | 基本持平（91.8→91.7） |
| patternList | 62.1% | 持平 |
| herbs | 98.6% | 98.5→98.6 |
| formulaList | 71.6% | — |
| treatmentList | 40.0%（2/5，样本极小） | — |

归上病因 top：`风湿`(10322)、`湿热`(4994)、`风寒`(1976)、`寒湿`(1366)、`风热`(416)、`外伤`(190)、`暑湿`(20)、`劳倦`(2)。

## 剩余 11576 条未归一病因：真实词表缺口
`饮食`(3456)、`气虚`(2609)、`血虚`(2253)、`阴虚`(1691)、`外感`(802)、`阳虚`(657)、`情志`(86)；另有 `风湿`(9)、`湿热`(8) 等零星残留。

结论（判断）：
- `气虚/血虚/阴虚/阳虚/外感` 是标准中医病因术语，`causes.json` 未收录 → **补词即可归上，确定性收益**。此时「主因（scope）已修完，剩下是真正的词表缺口」，补词统计才首次可信。
- `饮食`(3456) 需先甄别：大概率是抽取把问诊词根「饮食」误标成病因，补进病因词典前要确认来源，否则会固化错映射。

## 状态
- 「仅归一」已完成，**未重抽**（用户明确指示先不重抽）；方案A inquiry 抽离对存量的作用仍待全库重抽才能体现。
- 报告修复提交 `81a37f5` 已提交未推送（详见文档 `a7bc96bb-6d71-46d4-a658-dfe84122d591`）。
- 下一步：看质控合格率变化前**必须先清 Redis `tcm:ehr:cache:stats:report:*`**，否则读到 30 分钟内的旧报告（同见上述文档的 SWR 教训）。

## 排查经验（可复用）
- 判定「批任务是否真跑完」要看任务终态 + 同一分母下的各字段归一率，不能凭「用户说完成了」；同时核对 failed 明细的原因是否均为永久失败。
- 直接调后端接口时 `mvn`/HTTP 轮询易触发 600s 执行超时；本轮改为由用户/独立步骤确认完成后用一次查询取终态。

## 关联文档
- `01b35093-502c-4d8f-8334-a88c8dacc23b`：cause 0% 归一根因与修复顺序（本文是其第 2 步的实测闭环）。
- `a7bc96bb-6d71-46d4-a658-dfe84122d591`：报告「最该先补」过滤 + SWR 缓存掩盖代码变更。
- `21e3749c-b2c9-47df-9e32-99082757712e`：词典状态核实、inquiry 抽离与仅归一/重抽决策。

## 来源
整理自已完成的 DSH 委托维护检查点内的工具输出（NlpController/NlpTask 源码片段、批任务 JSON 提交响应、ES 归一率统计脚本结果），非本轮新查询。仓库根 `D:\ZISHIKU\AI\tcm-ehr-qc`。
