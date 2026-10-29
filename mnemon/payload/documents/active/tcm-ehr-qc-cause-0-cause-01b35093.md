---
id: "01b35093-502c-4d8f-8334-a88c8dacc23b"
title: "tcm-ehr-qc 病因 cause 0% 归一根因：归一范围不含 cause；「只归一不评分」双开关设计已存在"
description: "qc_rules 0行→出厂默认 8 类不含 cause→scope 整字段跳过；cause 已在 EntityTypes 注册可进 scope；normalization 与 standardization 两开关分离使 tongue/pulse/treatment 已实现只归一不计分，cause 同待遇零代码改动；词典重建≠重归一，仅归一批跑 8 分钟可救 cause；附用户三条问题清单的实测反证与修复顺序。"
status: "active"
created_at: "2026-10-20T02:07:00.015Z"
updated_at: "2026-10-20T02:07:00.015Z"
content_hash: "1ed1528e0abb8f55819538aeb48b1ac70766c8e50ac87383e744bb93edfa5daa"
source_paths:
  []
session_ids:
  - "68d23348-b849-44c7-a50e-149a1aa4b3ce"
memory_body_ids:
  []
---

# tcm-ehr-qc 病因(cause)0% 归一根因核实 + 归一/计分双开关设计已存在

## 结论先行（核实日期 2026-10-20，来源：委托检查点内只读诊断）
**病因 causeList 30862 条、0% 归一，根因不是「词表未建全」，而是归一范围(scope)不含 cause —— 质控规则配置缺口。**
且「字段少的实体只归一、不评分」这个设计**代码里已经实现**，无需改代码即可达成，只需让 cause 进 `normalization.elementTypes`。

## 根因链条（三层证据）
1. `qc_rules` 表 **0 行** → 全库没有任何组织配置过归一范围，全部走 `QcRuleSet.defaults()` 出厂默认。
2. `QcRuleSet.java:69-70`：`DEFAULT_NORMALIZATION_TYPES = List.of("disease","pattern","symptom","herb","formula","tongue","pulse","treatment")` —— **8 类，不含 cause**；类注释 line 266 自述「取 DEFAULT_NORMALIZATION_TYPES（8 类，不含病因）」。
3. `EntityNormalizer.normScope()`（`EntityNormalizer.java:61-63`）读 `ruleStore.normalizationTypes(orgId)` → 缺配置时退回上述 8 类默认 → 归一器按 scope 过滤时**跳过 causeList 整个字段**。

**反证（决定性）**：`term_cause` 词典里 `风湿/风寒/寒湿/湿热/痰湿/血瘀` 都是已收录的精确标准词，但 30862 条 causeList 实体 0 命中。若只是词表不全，命中的词应归上；**0% 只能是 scope 整字段跳过**。词表不全（`血虚/阴虚/阳虚/气滞/肝郁/肾虚/气虚` 未收录）只是次要问题。

## cause 有资格进 scope
`EntityTypes.java:74` 已注册 `new EntityType("cause", "病因", "causeList", true, "causes.json", …)`，key=`cause`、structuredKey=`causeList`、有词典文件；注释 line 16 说明 `dictKeys()` 现为 9 类。**cause 完全可被选进 scope，只是默认没勾它。**

## 「只归一不评分」设计已存在（两个独立开关）
- 归一范围：`EntityNormalizer.normScope()` → `ruleStore.normalizationTypes(orgId)`（读 `normalization.elementTypes`）
- 计分范围：`QcScorer.normalizedTypesOf()`（`QcScorer.java:274-281`）读同一份 `normalization.elementTypes`，但只与 `standardization.elementTypes` **取交集**
- `QcScorer.java:81-89` 交集逻辑注释自证：未纳入归一范围的类型不计扣分，避免「恒判未归一并扣满」的假信号
- 出厂默认（`QcRuleSet.java:252-260`）：`standardization.elementTypes = [disease, pattern, symptom, herb, formula]`（5 类）；`normalization` = 8 类（不含 cause）

→ **现状 tongue/pulse/treatment 已经是「只归一、不计分」**。只要让 cause 进 `normalization.elementTypes`，它就自动获得同等待遇，完全符合「字段太少的实体只做归一不做评分」的诉求，**零代码改动**。

## 第二层问题：词典重建 ≠ 重新归一
`rebuild ES` 只更新索引，已落库的 `structured_data` 是旧归一口径的产物，不会因索引更新而重算。`EntityNormalizer.normalizeStored(r, orgId, dictMeta)` 走「读存量 JSON → `entityNormalizer.normalizeMap(data, orgId)` → 写回」的全量归一（含 causeList），因此**「仅归一」批跑即可救 cause，约 8 分钟，不需要全库重抽（约 1h）**。

## 对用户三条问题清单的核实结论（数据实测）
| # | 用户判断 | 核实结果 |
|---|---|---|
| 1 | 病因已抽到实体却没归上，属词表未建全 | ❌ 主因是 scope 不含 cause（配置缺口）；词表不全排第二位 |
| 2 | 症状最该先补 8 个高频词（上下楼梯特别疼 15 次等） | ⚠️ 实测 symptoms 未归一 top30 里这 8 个词**一个都没出现**；top 全是问诊整句/脉象碎片：`大便日一行`(469)、`大便成形`(410)、`大便一日一次`(352)、`不受按`(228)、`红点多`(225)、`小便色略黄`(216) |
| 3 | 10 条 2 字未归一属词表未收录，补即可归一 | ❌ 实测 2 字未归一共 **9783 条、去重 2130 词**，top 是 `不痛`(145)、`夜寐`(93)、`俯卧`(87)、`浑浊`(85)、`仰卧`(82)、`温饮`(77)、`刷牙`(57)、`挂厕`(55)，多数不是症状而是问诊碎片/体位/脉象手法，补进症状词典会是错映射 |

补充：`symptomList`/`diseaseList` 查询返回 0 条是**字段名用错**，structured_data 顶层真实键为 `symptoms`/`diseases`/`causeList`/`patternList`/`tongueList`/`pulseList`/`formulaList`/`treatmentList`/`herbs`（另有 `_meta`/`truncated`/`modelAvailable`）。真实归一率：pattern 62.1%、tongue 75.9%、pulse 91.8%、herbs 98.5%、cause 0%（另有一次脚本漏写 `else: norm += 1` 分支导致全字段假 0%，以自洽数据为准）。

## 已确定的修复顺序（判断，非已执行）
1. 让 cause 进归一范围：改 `QcRuleSet.DEFAULT_NORMALIZATION_TYPES`（影响所有组织），或更稳妥——给当前组织插一条 `qc_rules` 显式勾选 9 类归一 + 5 类计分（不动出厂默认）。
2. 跑一次**仅归一**（约 8 分钟），让存量 structured_data 按新 scope 重算，causeList 拿到 normLevel。
3. **补词排最后**：当前 30862 未归一是脏数（scope 跳过导致），仅归一跑完后的剩余缺口才是真实补词依据；在存量仍是旧口径 + symptoms 混着问诊整句时统计高频未归一会补错。方案A inquiry 抽离代码已就绪但存量未重抽，故 symptoms 分母仍脏。

## 关联文档
参见文档 `21e3749c-b2c9-47df-9e32-99082757712e`（词典状态核实：豆包 8822→8860 落库/reindex、inquiry 抽离与精确回拉程序侧现状、仅归一 vs 全库重抽决策）。
