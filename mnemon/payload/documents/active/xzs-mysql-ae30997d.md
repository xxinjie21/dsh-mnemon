---
id: "ae30997d-5507-4bcf-8d5d-34f6e735017f"
title: "xzs-mysql 作为慢病科普答题系统底座：真实表结构核实与改造方案审核结论"
description: "学之思 xzs-mysql 项目表结构权威核实（t_question 无备注/标签字段、无错题本表、13 表清单）、对乙方改造方案的五项需求可行性审核、三处必改项与四个候选开源项目选型对照。"
status: "active"
created_at: "2026-10-21T15:52:43.447Z"
updated_at: "2026-10-21T15:52:43.447Z"
content_hash: "91eeab7a47d533876ea7d706eff2e52c48eec3627831f3ae7c9b4878b30976dc"
source_paths:
  - "sql/xzs-mysql.sql"
  - "source/xzs"
  - "source/vue"
  - "source/wx"
session_ids:
  - "2c4c09ad-e34c-4587-a452-3f6b83278d5c"
memory_body_ids:
  []
---

# xzs-mysql 底座选型与改造方案审核（2026-10-21）

## 背景

用户要做一套面向老年人的慢病科普答题应用，需求五项：题库、刷题学习、测试闯关、智能练习（按知识点出题）、谣言辨析（标记错误处）。已有一份乙方/外部方案建议以 Gitee `mindskip/xzs-mysql`（学之思开源考试系统 MySQL 版）为底座改造。本次工作是对该方案做事实核实与可行性审核。

## 权威事实：xzs-mysql 真实结构

GitHub `mindskip/xzs-mysql` 与 Gitee 同名仓库为同一作者；审核时 stars=3606，lang 标为 JavaScript（实际 Java+Vue+小程序），push=2026-10-21，License **AGPL-3.0**，未归档。

仓库结构：`source/xzs`（后端 Java）、`source/vue`（Web 前端）、`source/wx`（微信小程序）、`sql/xzs-mysql.sql`（12,727 B）、`docs/`、`docker/`、`release/`。

**方案说“删除项目内 miniprogram 文件夹”是错的，实际路径是 `source/wx`。**

全库 13 张表：
`t_exam_paper`、`t_exam_paper_answer`、`t_exam_paper_question_customer_answer`、`t_message`、`t_message_user`、`t_question`、`t_subject`、`t_task_exam`、`t_task_exam_customer_answer`、`t_text_content`、`t_user`、`t_user_event_log`、`t_user_token`。

`t_question` 建表语句（从仓库 sql 目录下载原文核实）：

```sql
CREATE TABLE `t_question`  (
  `id` int NOT NULL AUTO_INCREMENT,
  `question_type` int NULL DEFAULT NULL,   -- 1单选 2多选 3判断 4填空 5简答
  `subject_id` int NULL DEFAULT NULL,
  `score` int NULL DEFAULT NULL,           -- 千分制
  `grade_level` int NULL DEFAULT NULL,
  `difficult` int NULL DEFAULT NULL,
  `correct` text,
  `info_text_content_id` int NULL DEFAULT NULL,  -- 题干/解析/选项以 JSON 存于 t_text_content
  `create_user` int NULL DEFAULT NULL,
  `status` int NULL DEFAULT NULL,
  `create_time` datetime NULL DEFAULT NULL,
  `deleted` bit(1) NULL DEFAULT NULL,
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB;
```

关键结论：
- **没有 remark / tag / label 等任何标签或备注字段**，也没有可放标签的 varchar 列。题干、选项、解析全在 `t_text_content.content` 里，通过 `info_text_content_id` 关联。
- **没有错题本表**。错题只能由 `t_exam_paper_question_customer_answer.do_right = 0` + `create_user` 推导，是一段查询而非一张表。
- **没有关卡表**（需新增 `t_challenge`）。
- `t_exam_paper.paper_type`：1 固定试卷、4 时段试卷、6 任务试卷——固定试卷可重复练习成立。
- 试卷已有 `t_exam_paper_answer`（`create_user`、`exam_paper_id`、`system_score`、`user_score`、`status`、`do_time`），通关判定一次 SELECT 最高分即可。
- `t_subject` 有 `level` / `level_name`（设计为 K12 年级 1-12），非 K12 学科塞进去语义不合适但可用。

## 五项需求审核结论

| 需求 | 可行性 | 核心问题 |
|---|---|---|
| 题库 | 可行 | 后台录入即可，零代码；但不能用“备注字段”做分类 |
| 刷题学习 | 可行 | `paper_type=1` 固定试卷，建卷即用，零代码 |
| 测试闯关 | 部分可行 | 建表+接口没问题；用 localStorage 存进度是硬伤 |
| 智能练习 | **原方案不可行** | 依赖不存在的 remark 字段，SQL 写不出来 |
| 谣言辨析 | 部分可行 | 判断题可做，但“标记错误处”无对应设计，退化成普通判断题 |

## 三处必改项（其余部分原方案可照用）

1. `ALTER TABLE t_question ADD COLUMN tag varchar(50) NULL COMMENT '知识点标签';` —— 替代不存在的“备注”。`difficult`(1-5) 无法表达“饮食/用药”这类语义，必须加列。
2. 闯关进度放弃 localStorage，改用 `t_exam_paper_answer` 查最高分。方案自身矛盾：既要写“通关判定接口”又建议前端 localStorage，且答辩时“换机/清缓存进度归零”无法解释。
3. 谣言“标出错误处”需补设计：解析结构化标注，或题干分段按段判断。
4. 删除路径纠正为 `source/wx`。

## 工作量与风险提示

真实工作量大于原方案估计：DB +1 表(`t_challenge`) +1 列(`t_question.tag`)；后端 3 个接口（关卡列表/通关判定/智能练习）；前端 3 页 + 答题组件改造（智能练习需脱离试卷记录，`paper_answer` 强耦合，非纯复用）；CSS 适老化全局改造（Element-UI 学生端响应式但**无适老化设计**，组件样式覆盖麻烦，工作量被低估）。

**最大耗时不在开发，而在题目内容**：医疗题目不可用 AI 生成，需照临床指南人工编写，工作量大概率超过全部开发工作。

## 开源替代项目选型对照（2026-10-21 核实）

| 项目 | Stars | 语言 | License | 结论 |
|---|---|---|---|---|
| mindskip/xzs-mysql | 3606 | Java+Vue | AGPL-3.0 | 唯一同时满足技术栈匹配、有许可证、数据模型完整 |
| kesixin/dati | 383 | JS | 无 LICENSE | 毕设有版权风险；根目录为小程序代码，后端另需获取 |
| mawoka-myblock/ClassQuiz | 732 | Svelte | MPL-2.0 | 技术栈完全不同，非 Java，无法用 Java 答辩 |
| nyx-1037/ql-tiku2-opensource | 4 | Vue | MIT | 几乎无人用，且是 AI 单体架构，与“不用大模型”冲突 |

结论：**xzs-mysql 仍是最优底座**，但其“考试系统”形态（为期末考试设计）对答题/刷题场景有额外负担，需按上面三处修正后落地。

## 工具与环境踩坑（本次操作）

- `web_fetch` 访问 `github.com` / `raw.githubusercontent.com` 报 `resolves to a non-public IP address`，被拦截；改用命令行 HTTP 请求 GitHub API 与 raw 文件可正常工作。
- Gitee 网页被登录墙挡住；OSCHINA 页面需 JS 渲染。
- GitHub API 偶发限流（4 个仓库批量查询全部失败），重试或分批即可。
- PowerShell 脚本中 `pushed_at` 已被解析为 DateTime，对其调用 `.Substring` 会报 `Method invocation failed`。
- CSDN「学之思数据库设计文档」(blog.csdn.net/a87922072/article/details/117357832) 可作表结构旁证，但最终应以仓库 `sql/xzs-mysql.sql` 原文为准。

## 后续待办

用户尚未确认方向。可展开的下一步：把三处修正写成可执行改造清单，或先只解决智能练习标签设计。用户已表态“方向正确，可用作底架来改造”。
