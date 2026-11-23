---
id: "e8c74467-03ae-40ed-b341-fce752a3655e"
title: "DSH desktop 插件选型：浏览器批注 web-annotate 首选、去重排除清单与宿主版本推断"
description: "desktop profile 插件选型调查：从 peer 推断宿主 0.2.2-rc.2+、现有 bundle 清单、web-annotate 对 sidenotel/paper-margin 的取舍、与已装插件重叠的排除项。未安装、未实测。"
status: "active"
created_at: "2026-11-23T09:47:17.667Z"
updated_at: "2026-11-23T09:47:17.667Z"
content_hash: "f0f4ed83930cccf7a0c93a96dd9ea0763f25b9ce3ec475c4fcaa618cf3e4bde7"
source_paths:
  []
session_ids:
  - "mnemon-review-v1-d854e1ef-8218-48c0-afee-5e6e8392964c"
memory_body_ids:
  []
---

# DSH desktop 插件选型调查（浏览器批注 + 日常效率）

## 任务与状态
用户问「选哪一个」浏览器批注插件，以及「还需要装什么类似的插件来方便日常工作」。本轮只做搜索与推荐，**没有安装任何插件、没有重载验证、用户未做最终决定**。文中兼容性均来自公开 README / npm registry，未实装验证。

## 宿主版本推断（证据）
- `dsh-better-sidebar` v0.32.1 的 peer 全是 `@deepseek-ai/*: ^0.2.2-rc.2`，据此推断宿主为 **0.2.2-rc.2+ 系列**。
- `dsh-builtin-browser` v0.5.8：engines node>=22.19、dsh >=0.1.1-rc.2 <0.3.0。
- `billion-context` v0.1.216：engines node>=20。

## 现有 desktop profile 清单（C:\Users\xxinj\.dsh\profiles\desktop\package.json）
bundles（16 个）：@deepseek-ai/dsh-base、@deepseek-ai/dsh-web-app、dshmarket、@opencode2dsh/dsh-plugin、dsh-builtin-browser、dsh-mnemon、@hyzyn/dsh-codegraph、@michengai/dsh-btw、@michengai/dsh-code-review、@michengai/dsh-pua、@michengai/dsh-skills-manager、dsh-better-sidebar、dsh-ops、@michengai/dsh-archive-manager、@michengai/dsh-codex-ui、billion-context。
关键已装版本：dsh-codegraph ^0.6.22、archive-manager 1.4.1、btw 1.4.2、code-review 1.0.14、codex-ui 2.4.3、pua 1.0.13、skills-manager 1.5.0、opencode2dsh 0.3.14、billion-context ^0.1.216、better-sidebar 0.32.1、builtin-browser 0.5.8、dsh-mnemon 0.5.52、dsh-ops 0.5.5、dshmarket 1.78.0。

## 浏览器批注选型：首选 dsh-plugin-web-annotate
- `@wxlaz/dsh-plugin-web-annotate` 1.0.0（GitHub: uugvkbvkdjbsdbvwxxl）。就是「点选页面元素 → 写批注 → 高亮编号 → 攒一批发当前会话」的豆包式链路；以 extension 优先级**接管出厂 browser 标签**，不新增入口。package.json 只声明 `platform: web`，inject [dsh-client-ui-sidebar-right, dsh-client-ui-session]（用户已有）。
- 对比 `dsh-sidenotel-web`：偏元素检查器（发 selector/HTML/样式/截图），无写批注环节；新增标签不动出厂实现；声明 DSH >=0.2.1-rc.2 <0.3.0。对口度低一档。
- 对比 `dsh-paper-margin` v2.6.1（Aruvelut111）：侧栏划词批注 + PDF/Excel/Word/HTML + 侧聊，功能更全但偏「划词」；明确要 DSH 0.2.2 系列，兼容性信号最清楚。**npmjs 上 `dsh-paper-margin` 返回 E404，只能用 GitHub release tarball 安装。**
- 另有两个 `dsh-annotate` 是外部 Firefox/Zen 扩展 + 桥接，不在 DSH 右侧栏跑，已排除。

两个硬风险（推荐时已向用户点明）：① web-annotate 未写 engines.dsh / peerDependencies，desktop 能否干净加载需重载实测；② 它接管出厂 browser 标签，出问题等于整个内置浏览器不可用，回滚=卸插件。

## 日常插件：按「去重」而非「越多越好」
与已装 16 bundle 重叠、明确排除：
- `dsh-selection-toolbar`：/btw 侧问与已装 `@michengai/dsh-btw` 重叠。
- `dsh-diff-stat`：变更汇总/逐文件审查与已装 `@michengai/dsh-code-review` 重叠。
- `dsh-side-panel (ccq1)`：Git 评审+文件浏览器+终端与 better-sidebar + code-review 大面积重叠。
- `dsh-life-planner`：生活管理，非开发流。

值得加（按优先级）：
1. `dsh-select-quote`（latte03）：对话正文/文件预览划词批注+引用+就地翻译，正文留高亮和序号角标；0.2.2-rc.2+；侧边提问与 dsh-btw 重叠，装后关掉那个按钮。
2. `qiaomu-home-dsh`（乔木 Home，joeseesun）：会话搜索/今日重点/待办/快速记录/专注计时；只有市场页描述，未深挖 README；零重叠。
3. `dsh-code-quote`（hanrr92）：粘贴「路径:行号+代码」折叠成引用芯片省 token，已适配 better-sidebar「添加到对话框」；Web 输入框插件，desktop 加载未验证；零重叠。
- `dsh-paper-margin` 与 select-quote 批注场景重叠，二选一。

给出的闭环建议：web-annotate（网页点选）+ select-quote（文字划词）互补；想加每日入口再加乔木 Home。安装走 dep-upgrade-audit（备份清单→装→重载→验证→留回滚）。

## 其他事实
- 此前一次交付记录已写入 `feedback.jsonl`，只记可见 PUA 输出和任务摘要、无评分字段，范围是全项目。

## 相关文档（同一 desktop profile 环境）
- `5f77390e-3add-45eb-9bbc-11ee6d5549d1`：dsh-lsp-actions 安装后配置 + npm 全局 bin 不在 PATH 的根因与 profile 覆盖写法。
- `bf2bcf5d-39e1-4c5d-b005-0d8452bb1149`：dsh-mnemon 在 cordis.patch.yml 的配置最小化。
- `2b6eba09-014a-4c5b-ac9b-8b793f96bcaf`：glm-5.3 自动发 enable_thinking=false 导致百炼 400 的根因与修法。
