---
id: "21e3749c-b2c9-47df-9e32-99082757712e"
title: "tcm-ehr-qc 词典状态核实：豆包扩充词典(8822→8860)尚未生效于 ES/生效表 + 程序侧含 inquiry 抽离已就绪"
description: "三层层级一致证据表明豆包新词典未落库未 reindex；词典链路(reindex 端点/表列)与 inquiry 抽离+精确回拉程序侧 635 全绿的现状，及仅归一 vs 全库重抽的决策含义。"
status: "active"
created_at: "2026-10-19T17:39:15.473Z"
updated_at: "2026-10-19T17:39:15.473Z"
content_hash: "5b32d489e684271fe529e0117c2f36cf0f6ff8ab03aa89587d8c67bc5df6475a"
source_paths:
  - "docs/handoff-deepseek-harness-2026-10-20.md"
session_ids:
  - "00197647-7916-4ec9-a161-2bf537100397"
memory_body_ids:
  []
---

# tcm-ehr-qc 词典状态核实：豆包扩充词典尚未生效 + 程序侧(含 inquiry 抽离)已就绪

## 结论先行（核实日期 2026-10-20）
豆包那批「8822→8860 +38 标准词 / +196 别名词」词典**尚未落到当前生效表，也未推进任何 ES term_* 索引**。
因此此刻跑「仅归一」吃到的仍是**旧 477 条症状词典**，映射率不会涨。程序侧（含被问诊锚误吞症状的精确回拉兜底）已全部就绪且 635 测试全绿，但「程序完善 ≠ 词典到位」。

## 三层一致证据（核对当前生效渠道）
1. **MySQL dictionary_terms（当前生效表）**：按 org_id=''（全局默认桶 grp-default-2026）计数 symptom=477、herb=707、disease=1282、formula=2633、pattern=2146、pulse=125，与豆包表格的 535/711/1286/2634/2148… 逐项对不上。
2. **ES term_* 索引**：term_symptom=477、term_herb=707、term_formula=2633、term_pattern=2146、term_pulse=125、term_syndrome=30、term_disease=1282、term_tongue=103、term_treatment=1211。与当前生效表**逐项一致**，说明 ES 跟着当前生效表走、仍是旧版。
3. **dictionary_versions**：各类型 version=indexed_version（已索引），symptom 索引时间 2026-10-19 20:04:05，但条数仍 477 → 最近一次 rebuild 灌的还是旧表数据。
关键词佐证：当前生效表里「乏力/失眠」等已有，但豆包表格应新增的「大便成形」等没在。

## 词典链路（本次核实出的正确路径）
- 生效词典源表：**dictionary_terms**（列 id/org_id/type/standard_term/aliases/create_time/update_time —— 注意**没有 term 列**，标准词在 standard_term）。
- 版本表 dictionary_versions（org_id/type/version/indexed_version/indexed_at/update_time，**没有 id 列**）。
- 其他表：dict_archive_term（id/version_id/standard_term/aliases，归档层）、dict_proposal（提案层，id/type/status/audit…）。
- ES term_* 重建路径：**DictionaryServiceImpl.reindex → esTermIndexService.rebuild(type, orgId, entries, version)**，从当前生效表灌进 ES。对外端点 **POST /api/dictionary/reindex**。
- 关键判断规则：只改词典 → 仅归一即可（约 8 分钟），不用重抽；改抽取层 → 需重抽。

## 程序侧现状（inquiry 抽离方案 A）
- **python-nlp 提取/路由层**：_reroute_inquiry（问诊整句→inquiry）、_is_inquiry 谓词、schemas 加 inquiry 字段、main.py 接入 —— 验证 17/17 真问诊锚命中、0/8 反例误伤、幂等两次一致。
- **Java 引擎/持久化层**：VO/EntityNormalizer 加了 inquiry 透传（transport-only，不进归一/计分）；新增 **pullExactSymptomBack**：把被问诊锚误吞的**症状词典精确标准词**（嗜睡/乏力/便秘…）拉回 symptoms 参与归一 —— 只认 LEVEL_EXACT，绝不用含/模糊（避免「大便成形→大便溏」类错映射）。mvn 全量 **635 通过 0 失败**。

## 对决策的含义（判断，非已执行）
- 想让「inquiry 抽离、清空症状分母」在存量 34154 份病历上体现，**必须全库重抽**（python-nlp 重跑抽取才产生 inquiry 字段）；仅归一只吃「已有 structured_data」重算词典，不会补 inquiry。
- 要拿到「新词典 + inquiry 抽离 + 精确回拉」三项完整收益，正确顺序：**豆包词典先落库并 reindex（dictionary_terms → POST /api/dictionary/reindex）→ 验证 ES 条数=8860 → 全库重抽（约 1h）**。
- 工作区已含豆包 8 个 dictionaries/*.json（causes/diseases/formulas/herbs/patterns/pulses/symptoms/tongues/treatments），但未落库到当前生效表。

## 来源与局限
本记录来自 DSH 委托维护检查点内已存在的工具输出（git status 前的字典核实段），非本次新查询；文件路径均为仓库相对路径（仓库根 D:\ZISHIKU\AI\tcm-ehr-qc）。豆包实际落库动作由谁执行、文件落在哪一路径，检查点内尚未核实到，属遗留待办。
