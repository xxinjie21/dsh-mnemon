---
id: "a7bc96bb-6d71-46d4-a658-dfe84122d591"
title: "tcm-ehr-qc「最该先补」清掉问诊整句：INQUIRY_LEAD 过滤 + SWR 缓存掩盖代码变更的排查教训"
description: "报告字典缺口 top 清单被问诊整句污染 → 加 INQUIRY_LEAD 前缀过滤并锁定测试；排查中定位 StatsCache SWR 实际存储 TTL = REPORT_TTL×10(30min)、不随重启失效，验证报告类改动必须先清 tcm:ehr:cache:stats:report:*；附两字桶实测构成与前端文案结论。"
status: "active"
created_at: "2026-10-20T02:41:34.529Z"
updated_at: "2026-10-20T02:41:34.529Z"
content_hash: "ed4addd6371a90f93516b7d4eae301d3680a2ad3c1a3fdeecae1fbfa232f0ac2"
source_paths:
  - "java-backend/src/main/java/com/tcm/ehr/service/impl/StandardizationReportServiceImpl.java"
  - "java-backend/src/test/java/com/tcm/ehr/service/StandardizationUnmatchedTopTest.java"
  - "frontend/src/views/StandardizationReport.vue"
  - "python-nlp/nlp_rules.py"
session_ids:
  - "58a3a6ea-a1a4-4dbf-8560-5e837d331bb5"
memory_body_ids:
  []
---

# tcm-ehr-qc「最该先补」清掉问诊整句：过滤实现与 SWR 缓存排查教训

## 结论
标准化报告里「最该先补」的词表缺口 top15 被问诊整句污染（`大便日一行` 469、`大便成形` 410、`大便一日一次` 352、`大便日` 235、`小便色略黄` 216、`饮水多` 175、`食量一般` 168…）。加入 `INQUIRY_LEAD` 前缀过滤后，top15 变为真实缺口（`不受按` 228、`红点多` 225、`咽喉有痰` 163、`两手指端略自温` 132、`中取不受按` 115、`手心较热` 80…），问诊整句残留 **0**。
**统计口径完全不变**：`total=125910`、`fragment=9743`、`physicalSign=700`、`misrouted=2`、`dictionaryGap=115465` —— 过滤只作用于可行动清单（`b.getTop()`），不改任何分桶计数。

## 实现
- `java-backend/.../service/impl/StandardizationReportServiceImpl.java`：新增 `private static final String[] INQUIRY_LEAD`（原 18 个问诊词根 + 新增 11 个 = 29 个）。原有：`大便 小便 二便 排便 夜寐 睡眠 入睡 饮食 食欲 饮水 食量 纳食 胃口 精神 面色 乏力 倦怠 困倦`；新增时段前缀 `晨起 晨间 清晨 夜间 夜里 睡前 醒后 饭后 食后 平素` + 喜恶前缀 `喜`。
- `unmatched()` 的 `else`（长度 > `FRAGMENT_MAX_LEN`=2 的 dictionaryGap 分支）：`b.setDictionaryGap(+1)` 照旧统计全部，但 `gapCount.merge(content,…)` 外包一层 `if (!startsWithAny(content, INQUIRY_LEAD))`；排序仍走 `gapCount` 降序 `limit(15)` 装入 `b.getTop()`。
- 用**前缀**匹配而非包含匹配：`晨起小便淡黄`/`夜间明显` 剥掉时段后仍以问诊词根开头；包含匹配会误伤真症状（如含「喜」的体征词）。
- 单字锚 `喜` 有误伤风险（`喜叹息` 属问诊喜恶，`喜按` 属体征），代码中以注释说明取舍。
- 口径与 `python-nlp/nlp_rules.py#_INQUIRY_LEAD` 同源（方案A），但 Java 侧只用于**报告排序过滤、不改数据**，故取更长的一组避免误伤。
- 新增测试 `StandardizationUnmatchedTopTest.java#topExcludesInquirySentences` 锁定行为。
- 全量 `mvn -o clean test`：**637 通过 0 失败**；前端 `vite build` 成功。

## 关键教训：SWR 缓存会掩盖代码变更（本次排查的最大陷阱）
现象：改完代码、`mvn compile` exit=0、重启后端（class mtime 10:34:44 < 进程 StartTime 10:34:50，`javap` 确认 class 中 `INQUIRY_LEAD` 长度为 29 且 `startsWithAny` 调用在字节码中存在），接口返回的 top 仍然带 `晨起小便淡黄` 等；**等 185s 缓存过期后结果依旧不变**，据此一度误判为「代码没生效」。

根因（`StatsCache.getOrLoadSwr`，见类注释 51-55 行）：值存 `dataKey`，**存储 TTL = 基准 TTL × 10**，另用 `dataKey:fresh` 作新鲜标记（TTL = 基准）。标记过期后仍**立即返回旧值 + 后台单飞重算**。所以 `REPORT_TTL=180s` 的旧报告在 Redis 里活 **1800s（30 分钟）**；且版本号只在**写入**时自增（`invalidateForWrite()`），**重启后端不变版本号** ⇒ key 不变 ⇒ 一直返回旧报告。

证据：`RedisKeys.PREFIX + cache:stats:report:v739:…` 找到 4 个键，`ttl=1761`（≈29 分钟），删除后重算即得正确结果。

**规则：验证任何报告/统计类改动，必须先清 `tcm:ehr:cache:stats:report:*`（必要时连同 `:fresh` 标记），不能靠重启或等待短 TTL。**

## 顺带确认的第二个问题：两字桶文案是错的
实测 9743 条 2 字未归一实体：单字残字 **0 条**；体位描述 2.2%、问诊词根碎片 2.5%、否定式/属性词 3.7%，其余 91.3% 是 `浑浊/偏黄/淡黄/凌晨/刷牙/唇暗` 这类颜色性状词与时间场景碎片 —— **不是可批量登记的规范症状词**，补词会导致错映射。据此重写 `StandardizationReport.vue` 的 fragment 文案（删掉旧高频词列表与「补进症状词典即可归一」结论），标题改为「两字未归一实体需人工研判」，并在 Java 侧同步修正该桶的陈旧注释。

## 对用户清单第 2 条的修正
用户列的 `上下楼梯特别疼`(50)、`遇劳加重`(48)、`脾气很急`(39) 数字准确，但它们在 `dictionaryGap` 中排名 84-152，而页面只取前 8；过滤生效后这些词会正常进入可见清单。

## 遗留（检查点内未完成）
- 改动**尚未提交**（用户要求「只提交，不推送」，提交动作在检查点结束时仍在等待确认）。涉及文件：`StandardizationReportServiceImpl.java`、`StandardizationUnmatchedTopTest.java`、`StandardizationReport.vue`。
- `晨起小便淡黄`/`喜趴睡` 等问诊整句仍计入 `dictionaryGap` 计数（仅从 top 清单排除），存量 `symptoms` 里的问诊句子要靠 python-nlp inquiry 抽离 + 全库重抽才能根治。

## 关联文档
- `01b35093-502c-4d8f-8334-a88c8dacc23b`：cause 0% 归一根因 + 用户三条清单的实测反证（本文修正了其中第 2 条的结论）。
- `21e3749c-b2c9-47df-9e32-99082757712e`：词典状态核实与 inquiry 抽离程序侧现状。

## 来源
本记录整理自某个已完成的 DSH 委托维护检查点内的工具输出（含 grep/javap/redis-cli/mvn test 的实际结果），非本次新查询。仓库根 `D:\ZISHIKU\AI\tcm-ehr-qc`。终端为 GBK 编码，`javap` 输出中文显示为乱码但不影响按常量索引计数判断。
