---
id: "2b8d01cc-cf0b-499b-a070-74dbd7a8e636"
title: "tcm-ehr-qc 注释规范整改落地：{@code}/HTML 标签清理口径与验证基线（2026-10-24）"
description: "14 批注释规范整改的最终口径（{@code} 仅用于标识符、注释禁用 HTML 标签改空行分段与 - 列表、Controller 权限标注、类注释三段式）、925+3465 清理量、637 测试与 34154 冒烟基线，以及统计误报与历史遗留单行 Javadoc 的排查教训。"
status: "active"
created_at: "2026-10-23T16:27:11.728Z"
updated_at: "2026-10-23T16:27:11.728Z"
content_hash: "3f899c565b7d8bbf8cdec08fb19f077b4cb8fd18a4dee01a3fcd0e2f0b098f5f"
source_paths:
  - "java-backend/src/main/java/com/tcm/ehr/common/config/QcRuleSet.java"
  - "java-backend/src/main/java/com/tcm/ehr/common/config/EntityTypes.java"
  - "java-backend/src/main/java/com/tcm/ehr/common/cache/StatsCache.java"
  - "java-backend/src/main/java/com/tcm/ehr/common/utils/LlmClient.java"
  - "java-backend/src/main/java/com/tcm/ehr/service/impl/QcServiceImpl.java"
  - "java-backend/src/main/java/com/tcm/ehr/service/impl/StandardizationReportServiceImpl.java"
  - "java-backend/src/main/java/com/tcm/ehr/service/impl/StatsServiceImpl.java"
session_ids:
  - "f9e4c534-77b1-4f39-a11e-4fa3ec762e87"
memory_body_ids:
  []
---

# tcm-ehr-qc 注释规范整改落地记录（14 批，2026-10-24）

## 背景与结论

对 `tcm-ehr-qc` 仓库按项目文档规范（规范 2 / 3 / 11）做的一次全库注释整改，拆成 14 个可独立回滚的 commit（`680a14d` … `2f6154d`），全部已推送、本地与远程同步 0/0。

最终指标（整改前 → 后）：

| 指标 | 前 | 后 |
|---|---|---|
| HTML 标签（`<p>`/`<b>`/`<ul>`/`<li>`/`<br>`） | 3465 | 0 |
| `{@code}` 套普通标识符 | 925 | 0（保留 636 处合理用法）|
| Controller 方法缺【权限】标注 | 18（占 85 个方法）| 0 |
| 类级 Javadoc 缺失 | 9 | 0 |
| 前端错误分节标记 | 6 | 0 |
| `SCORE_BUCKETS` 双份定义 | 1 组 | 0 |

## 已落地的规范口径（后续新增代码照此写）

- **`{@code}` 只用于需要等宽呈现的标识符**（方法名、字段名、类名、注解名）。关键字（`null`/`true`）、表达式、SQL 片段、字面值**保留壳**——去壳会让这些内容在 Javadoc 里失去等宽与转义保护。全库最终保留 636 处即按此判据。
- **注释里不使用 HTML 标签**。原来的 `<p>…</p>` 段落改为**空行分段**（这是关键：直接删标签会把相邻段落粘成一行）；`<ul>/<li>` 改为 `- ` 列表，跨行 `<li>` 的续行保持原有缩进；`<b>` 只删标签保留文字，**不**改写成「注意：」类前缀（避免凭空制造数百个并不存在的"重点"）。
- **Controller 每个方法都要有【权限：…】标注**，分档写法与 OrgController 现状一致（匿名 / 登录即可 / 仅管理员 / 所有者（本组织））。
- **类级 Javadoc 三段式**：概述 → 动机 → `⚠️` 风险/边界。
- **前端分节注释**统一为 `// ===== 名称 =====`。

## 值得复用的做法与教训

1. **先分析形态再批量替换**：`<p>` 统计出 4 类形态（仅开头 799 / 仅结尾 797 / 单行 141 / 同段 25），据此定策略；先单文件试点（`JwtUtil.java`、`EntityTypes.java`）看输出再全库。
2. **统计口径必须先复核再宣告**，本轮自查修正了 3 次错误数字：类注释缺失 193→9、权限缺失 81→18、`logout` 误报。根因都是正则没跨过注解/多行注解被截断。**多行 `@RequireRole(...)` 必须跨行匹配**。
3. **批量文本改写后要扫"注释被压成一行"**：本轮在 `java-backend/src/main/java/com/tcm/ehr/service/impl/StandardizationReportServiceImpl.java:1055` 发现整段 Javadoc 变成 214 字符单行。用 `git show HEAD:<file>` 比对确认这是**改动前就存在的历史遗留**，不是本轮造成——排查前先分清归因，再重排为 7 行。
4. **每批都跑 `mvn clean test`** 再提交；文档类改动也不跳过（Javadoc 破坏会导致编译失败）。

## 验证基线（可作为后续改动的回归对照）

- `mvn clean test`：**637 通过**（14 批每批都跑）。
- 服务启动后冒烟：看板 合格=28801 / 待复核=1838 / 无效=3515，合计 **34154**；复核 pending=1838、overdue=0；清洗任务 17 条；报告「未归一」total=78244。
- 数据与改动前基线完全一致——说明纯注释/常量重构未影响业务输出。

## 涉及的主要文件（示例）

- 注释整改主战场：`java-backend/src/main/java/com/tcm/ehr/common/config/QcRuleSet.java`、`.../common/config/AsyncConfig.java`、`.../common/config/EntityTypes.java`、`.../common/cache/StatsCache.java`、`.../common/utils/LlmClient.java`、`.../common/utils/RecordFilter.java`、`.../common/utils/ExcelRawStreamReader.java`、`.../service/impl/QcServiceImpl.java`、`.../service/impl/QcBatchServiceImpl.java`、`.../service/impl/StandardizationReportServiceImpl.java`、`.../mapper/RecordMapper.java`。
- 逻辑侧两处小重构：`StatsServiceImpl` 的评分分桶（阈值抽 `BUCKET_MIN`，0-100 全分数比对差异 0）；LLM 侧 `flag(value, base)` → `resolveFlag(userValue, baseline)`（语义更明确：用户值优先、基线兜底）。

## 相关文档

- `50f46acc-fbb6-484f-a5e0-d01e8f010a16`：同一仓库 2026-10-23 的全量代码审查（含 P1/P2 可读性项，本次整改即其下游）。
- `eaa00c66-9d76-425b-b317-8e36f63fd1fd`：同一仓库的验收评审与 P0/P1/P2 整改清单（本轮的 P2 部分）。
