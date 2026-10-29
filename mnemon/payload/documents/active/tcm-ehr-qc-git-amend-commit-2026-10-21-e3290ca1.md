---
id: "e3290ca1-4db1-4a41-87d2-090a9a4ad9fe"
title: "tcm-ehr-qc「无人点击的批量解析」排查 + git amend 覆盖他人 commit 的恢复纪律（2026-10-21）"
description: "全库 EXTRACT 任务来源不明的排查：request_key/filters 指纹判据、6 类自动触发源逐一排除、并行会话证据链；git commit --amend 以暂存区重建导致他人 commit message 被覆盖的 reflog 恢复步骤；抽取层 vs 归一层的生效条件与耗时（重抽约1h / 仅归一约10-17min）。"
status: "active"
created_at: "2026-10-21T08:12:12.397Z"
updated_at: "2026-10-21T08:12:12.397Z"
content_hash: "382538231d432d280d2721c9c1d743f934e63a564c8f15d110e4967f0ec6841e"
source_paths:
  - "frontend/src/components/NlpBatchPanel.vue"
  - "frontend/src/api/nlp.js"
  - "java-backend/src/main/java/com/tcm/ehr/controller/NlpController.java"
  - "java-backend/src/main/java/com/tcm/ehr/service/impl/RecordServiceImpl.java"
  - "java-backend/src/main/java/com/tcm/ehr/common/utils/PythonNlpClient.java"
  - "python-nlp/nlp_filters.py"
  - "start-all.bat"
session_ids:
  - "cb58200d-ac65-48de-92a6-863a784c78ba"
memory_body_ids:
  []
---

# 「没人点的批量解析」排查复盘 + git amend 覆盖他人 commit 的恢复（tcm-ehr-qc，2026-10-21）

用户问：**「为什么重跑解析了，不是我按的」**。本文记录已排除的路径、定位到的证据链、一个被自己触发的事故（amend 覆盖），以及可复用的判据与恢复步骤。

## 一、定位判据（可复用：如何判断一个 NLP 批任务不是前端提交的）

后端 `nlp_task` 行上有三个字段足以做来源指纹：

| 字段 | 前端提交（当前代码） | 空参/非前端提交 |
|---|---|---|
| `request_key` | 每次点击都 `crypto.randomUUID()` | `None` |
| `filters` | 总是对象（如 `{onlyUnparsed:true}`） | `null` |
| `limit` | 0 或指定值 | 0 |

证据点：
- `frontend/src/components/NlpBatchPanel.vue:433-440`：payload 固定为 `{filters, limit, requestKey}`，再按 `isNormalize` 走 `submitNlpNormalizeBatch` / `submitNlpBatch`。注释说明幂等键的语义（网关重放沿用同一个 key）。
- `java-backend/.../controller/NlpController.java:176,202` 是 `submitBatch` 的两个入口；`@RequestBody(required = false)` 允许空 body → 空 body 就产生 `filters=null, request_key=None` 的全库任务。

所以「`request_key=None` + `filters=null` + total=34182（全库）」= **不是前端面板按的**。

## 二、逐项排除的其他自动触发源（均已查证）

1. **定时任务**：全项目 `@Scheduled` 计 0 个。
2. **前端自动触发**：`DictionaryImport.vue` 的 `rerunAll` 需点击（且上一轮已改为 `submitNlpNormalizeBatch`，只归一）；`NlpBatchPanel.vue` 只有 `watch(() => props.active)` 用于**停止**轮询（不启动），`onMounted` 只拉列表/启轮询。
3. `start-all.bat`（219 行，GBK + CRLF）：只负责起 mvn spring-boot:run 与 `python-nlp/.venv/Scripts/uvicorn.exe main:app --host 127.0.0.1 --port 8001`，**全文无任何接口调用**。
4. **导入后自动解析**：`RecordServiceImpl.java:134-142`，条件是 `autoExtract && !toInsert.isEmpty()`，且 total 会等于本次导入条数，不会是全库 34182。
5. **后端自动拉起 / 自动重试**：`PythonNlpClient` 只做 HTTP 调用；重启兜底只把 RUNNING 标 `INTERRUPTED`，不会重跑。
6. **我自己的临时脚本**：`%TEMP%/opencode/t_*.py` 里确实有多个提交 extract/batch 的遗留脚本，但当时运行的 python 进程是 `uvicorn.exe ... --port 8001`（NLP 服务本体，16:01:04 启动）与一个无关的 mcp 进程，**不是这些脚本**。

## 三、证据链：仓库里存在另一个并行操作者（强证据，未最终与人对质）

- reflog：`4f91900 HEAD@{1}: commit: fix(nlp): 中药黑名单补全 …` —— 该 commit 不是本方会话所做，且 message 含 47 个用例的实测细节，属于另一侧的在途工作。
- 时间线自洽：16:01:04 NLP 服务重启（早于本方任何动作）→ 16:03:25 本方 NORMALIZE_ONLY 任务被 `cancel_req=1` → 16:03:44 出现全库 EXTRACT → 16:09:19 又一次全库 EXTRACT，前一个随即 CANCELLED。
- 结论：**两个会话在同一工作树上并行跑批，后提交会取消前者**，导致谁都跑不完。要真正定论需向用户确认是否存在另一会话。

## 四、事故与恢复：git commit --amend 吞掉别人的 commit

**事故机制**：`git commit --amend` 以**当前暂存区**重建 commit。当时暂存区里混入了另一侧的 `python-nlp/nlp_filters.py`，于是 amend 把原 commit（本方 `EntityNormalizer` 改动）替换成对方的文件改动，并把 message 改成本方的。

**判定与恢复步骤（可复用）**：

1. 看 `git reflog`，定位被覆盖的原始 commit（本例 `4f91900`）。
2. 比对文件改动是否等价：`git show --stat <old>` 与 `git show --stat HEAD`。本例两者都是 `python-nlp/nlp_filters.py | 27 ++++---`、24 增 3 删，`git diff` 为空 → **代码一行未丢，只丢了 message**。
3. 只恢复 message：在暂存区与 HEAD 树一致时立即再 `git commit --amend -m "<原 message>"`（本例得到 `743e062`，内容不变）。
4. 事后核验：`git log -3 --stat` + 关键方法/符号计数（本例确认 EntityNormalizer 的 6 个关键点仍在，因为它已在 `040b5d2` 里）。

**纪律**：在有并行进程/会话的工作树上，**不要用 `--amend` 和 `git add -A`**；先 `git status` 确认暂存区，必要时用 `git commit -- <paths>` 精确指定文件。

## 五、改动生效的层次语义（决定「重跑归一」够不够）

| 改动在哪 | 生效条件 |
|---|---|
| Java 归一阶段（`EntityNormalizer` 等） | 重启后端即可；系统结构化数据（`structured_data` 是抽取时写的快照）**不受影响**，重跑归一即生效 |
| Python 抽取层（`python-nlp/nlp_filters.py` 黑名单词根等） | 需重启 NLP 服务 **且必须重抽**才生效 |

实测代价参考：全库 34182 条，重抽约 1 小时量级，仅归一约 10–17 分钟量级；归一进度可达 29096/34182 后被打断。

## 六、相关文件

- `frontend/src/components/NlpBatchPanel.vue`、`frontend/src/api/nlp.js`
- `frontend/src/views/DictionaryImport.vue`、`frontend/src/components/StandardizationDetails.vue`
- `java-backend/src/main/java/com/tcm/ehr/controller/NlpController.java`
- `java-backend/src/main/java/com/tcm/ehr/service/impl/RecordServiceImpl.java`
- `java-backend/src/main/java/com/tcm/ehr/common/utils/PythonNlpClient.java`
- `python-nlp/nlp_filters.py`、`start-all.bat`
- 相关工作树现状：`EsTermNormalizer.java` 有未提交修改；仓库里留有若干 `.dsh-ops-update-*.lock` 未跟踪文件。

## 七、未决问题（需用户裁决，本文不预设结论）

两个会话并行导致批任务互相取消；是停手让另一侧跑完，还是统一收尾（并接受为抽取层改动重抽一次的代价），需用户指定以哪一侧为准。
