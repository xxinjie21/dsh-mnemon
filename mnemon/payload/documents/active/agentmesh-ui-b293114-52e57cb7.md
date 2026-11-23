---
id: "52e57cb7-609d-4626-bc4e-f6401a7031b8"
title: "AgentMesh 底部对话框为主、终端可折叠为次（UI 主次收口 b293114）"
description: "输入框升为主操作区、终端降为默认收起的输出面板：结构、键盘语义、! 命令自动展开、真实浏览器验证与可复用经验；承接 f772c2e3 的根因与边界。"
status: "active"
created_at: "2026-10-29T17:51:19.058Z"
updated_at: "2026-10-29T17:51:19.058Z"
content_hash: "798bf5d9be15bf916081acb40baa58dd1bc3a6b1de91bf366b8a3004fbe818b7"
source_paths:
  - "ui/app.js"
  - "ui/index.html"
  - "ui/styles.css"
session_ids:
  - "75a6c307-ef99-4bc5-99b1-66d32677bfae"
memory_body_ids:
  []
---

# AgentMesh 底部对话框为主、终端降级为可折叠输出面板（UI 主次收口）

> 来源：工作区 `D:\ZISHIKU\AI\AgentMesh` 的 UI 主次修复迭代。提交：`5a22783`（primary composer, collapsible terminal）、`b293114`（DeepSeek-style chat composer, terminal secondary）。根因、接入成员面板、服务发现与更早的单一底部输入收口见 `f772c2e3-a8c2-4553-9945-ac9dfbd3f5bf`；本文只记录“输入为主体、终端为次要”这一层决策与验证，不重复前文。

## 1. 用户纠正与设计原则

用户两次递进指出：
1. “输入框应该为主体吧，命令行为次要地位”；
2. “你不能去采用 DeepSeek Harness 等的对话框吗”。

结论：问题不在输入框长短，而在**把聊天输入、终端输入、派活控件塞进同一条工具栏**，视觉上像命令栏而非对话框。修法是把派活条拆成两块——底部圆角对话输入区为主操作，终端只做输出。

## 2. 最终结构（b293114）

- **底部主体**：圆角输入区，2 行 `textarea`（实测高 46px），右侧发送按钮，占满窗口宽度，固定在 `.app` 最后（`main` 下方），底部与窗口底部对齐。
- **输入区上方一行元信息**：发给谁 / 马上执行·等空闲 / 等结果，小号控件，不再与输入争宽度。
- **终端**：默认收起，只保留标题栏；标题栏按钮“展开终端 / 收起终端”。可拖高度（手柄 `rzTerm`，默认曾为 236px，后降到 170px，最后改为默认收起）。
- **旧终端输入行**：`.term-pane .term-in` 隐藏，点击终端区域直接用 xterm 原生键盘输入。
- **键盘语义**：Enter 发送；Shift+Enter 换行；`!` 前缀执行终端命令；`/` 前缀是系统命令。

## 3. 行为细节

- 输入 `!echo CHAT_OK` 时**自动展开终端**并显示回显，命令仍复用同一个输入框，不另设终端输入行。
- 展开/收起终端会改变 `.main` 的 grid 行高，画布/看板随之上移；之前出现的“终端被压成 0 行”是 grid `44px 0px …` 的直接后果，需同时调 grid 行与 `--term` 变量。
- 输入框标签必须是 `TEXTAREA`（多行、可 Shift+Enter），不能用单行 input 冒充对话框。

## 4. 验证证据（真实浏览器，1280×800 / 1440×900）

- 输入框：`tag=TEXTAREA`、`textareaH=46`、`composerBottom=800` = 窗口高 800。
- 终端默认 `collapsed=true`；点击“展开终端”后 `termVisibleAfter=block`、按钮变“收起终端”；旧 term-in 为 `none`。
- `!echo CHAT_OK` 后终端自动展开并命中内容；console 报错为空。
- 全量测试仍为 118 pass / 0 fail / 2 skip（真机 e2e 默认跳过）。

## 5. 可复用经验

- 用户说“对话框和命令行冲突”时，先数**输入控件的数量与语义层级**：同一区域有 2 个以上输入控件、或控件平铺在一行，就会读成命令栏；主入口应只有一个多行输入区。
- 参考成熟产品（DeepSeek Harness 式）的形态比调宽高有效：圆角输入区 + 上方一行元信息 + 右侧发送。
- 终端作为“可折叠的次要输出面板”时，展开动作要由事件驱动（收到 `!` 命令结果），不要总是常驻。
- 主次关系要用实测尺寸和 DOM parent 证明（bottom 是否等于 innerHeight、tag 是否为 TEXTAREA），不能只靠截图观感。
