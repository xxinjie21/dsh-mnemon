---
id: "226fc62f-d2b7-4ba5-8532-b4e3f4154552"
title: "tcm-ehr-qc 西医诊断列漏回补修复 · EntityNormalizer（2026-10-20）"
description: "backfillDiseases 只扫 tcmDiagnosis 漏西医诊断列的真疾病（约1422条）；修复=新增带 westernDiagnosis 的6参重载、旧5参委托null保持测试兼容，3文件、637测试全绿；关键：仅抽取/重抽生效非仅归一。"
status: "active"
created_at: "2026-10-20T05:30:05.607Z"
updated_at: "2026-10-20T05:30:05.607Z"
content_hash: "26a6bd4f627d7e0c646e92665bf363f5ce15af5f0fcbdf4e792178f39d63e252"
source_paths:
  - "java-backend/src/main/java/com/tcm/ehr/common/utils/EntityNormalizer.java"
  - "java-backend/src/main/java/com/tcm/ehr/service/impl/NlpBatchServiceImpl.java"
session_ids:
  - "f6a439d4-be9b-4ee1-a3a7-bd96a2fa20c2"
memory_body_ids:
  []
---

# 西医诊断列漏回补修复（EntityNormalizer）

## 背景
质控判定「疾病」可由「西医诊断」或「中医诊断」满足，但 `backfillDiseases` 回补逻辑只扫描 `tcmDiagnosis` 列，**西医诊断列的疾病从未进入 `diseases` 字段**。

影响：约 1422 条「西医诊断=真疾病」的病历漏抽，其中约 60 条修完能跨过合格线。

## 根因
- `EntityNormalizer.backfillDiseases(vo, orgId, tcmDiagnosis)`（约 810 行）只用 `termNormalizer.scan("disease", orgId, tcmDiagnosis)`。
- `backfillFromRaw` 的两个重载都没有 westernDiagnosis 参数。
- `NlpBatchServiceImpl:1193` 调用点只传 `r.getTcmDiagnosis()`。

## 修复设计（保持向后兼容）
为避免破坏既有测试，采用「新增重载 + 旧重载委托 null」：
1. `backfillDiseases` 增加 `westernDiagnosis` 参数，把中医+西医两列拼起来调用 `scan`（复用现有 scan，不裸切分，避免把非标准词塞进 diseases）。
2. `backfillFromRaw` 新增带 `westernDiagnosis` 的组织级 **6 参重载**（唯一干活的重载）。
3. 旧 5 参重载委托 6 参（西医传 null），保持现有测试 `EntityNormalizerPatternBackfillTest` 不变。
4. `NlpBatchServiceImpl:1193` 改为调用 6 参，传 `r.getWesternDiagnosis()`。

## 改动文件
- java-backend/src/main/java/com/tcm/ehr/common/utils/EntityNormalizer.java
- java-backend/src/main/java/com/tcm/ehr/service/impl/NlpBatchServiceImpl.java（1193 行调用点）

## 验证
- `mvn clean test` **637 全绿、BUILD SUCCESS**（含旧 5 参兼容路径测试）。

## 关键生效场域（操作须知）
**这个修复只在「批量抽取/重抽」时生效，不在「仅归一」时生效。**
`backfillFromRaw` 是抽取流程里的回补步骤；「仅归一」只读存量 structured_data 重算 normLevel，不会再重跑抽取去回补西医诊断。要看到 1422 条西医保真疾病被补进 diseases，需对存量走**重抽/补抽取**。这与「吃新词典 + 程序修复」的统一重抽动作接轨。

## 关联背景
- 计划文档「方案B：抽取正确率——⑤ 程序缺陷：西医诊断列漏回补（待执行）」已更新为「已修复」。
- 其他抽取归位项（`_reroute_inquiry` 问诊整句归位）阻塞在「面色/进食/睡眠」形态的业务口径，未动。
