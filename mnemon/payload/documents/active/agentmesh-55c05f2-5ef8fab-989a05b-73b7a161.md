---
id: "73b7a161-dbaa-47c2-897f-a5f1a2605590"
title: "AgentMesh 三栏可收起布局：对话框与终端收敛到中栏（55c05f2 / 5ef8fab / 989a05b）"
description: "三栏网格为唯一布局所有者：终端与底部对话框只占中栏、终端收起隐藏横向手柄、左右侧栏可按钮/拖拽阈值收起；含根因、收起语义与真实浏览器实测；承接 52e57cb7 与 f772c2e3。"
status: "active"
created_at: "2026-10-29T17:59:20.253Z"
updated_at: "2026-10-29T17:59:20.253Z"
content_hash: "a4432b09bb39f25d327322a2dfc14565a67e04d27271772a8c8c7e4e512b01e6"
source_paths:
  - "ui/app.js"
  - "ui/index.html"
  - "ui/styles.css"
session_ids:
  - "85cf169b-4e23-4995-aa01-cefe5075100e"
memory_body_ids:
  []
---

# AgentMesh 三栏可收起布局：对话框与终端收敛到中栏

> 来源：工作区 `D:\ZISHIKU\AI\AgentMesh` 的 UI 布局迭代。相关提交：`989a05b`（align resizers with collapsed terminal）、`55c05f2`（keep terminal and composer inside center column）、`5ef8fab`（collapsible side panes with drag threshold）。
> 承接文档：`52e57cb7-609d-4626-bc4e-f6401a7031b8`（输入为主体、终端为次要的对话框层）、`f772c2e3-a8c2-4553-9945-ac9dfbd3f5bf`（单一底部输入、可拖分栏的初始收口）。本文只记录“布局归属与可收起”这一层，不重复前两篇。

## 1. 用户纠正与设计意图

用户在三次迭代中逐步收紧要求：

1. “有一个游离的用于控制窗口大小的拖动条”——终端收起后仍有悬浮分界线。
2. “命令行和对话框应该只处于中部的那一栏啊，现在软件分布是 3 栏左中右”——终端与对话框横跨整行，破坏了左/中/右三栏。
3. “3 栏左中右都要像市面上的软件一样能够收起，拖动条拖到一定程度就直接收起，可以采用 deepseekharness / opencode 的前端分布思维”。

结论：这三条不是尺寸微调，而是**布局所有权**问题——终端、对话框、拖拽柄各自的位置必须由同一套网格数据决定，侧栏必须真正做到可收起，而不是固定占位。

## 2. 根因

| 现象 | 根因 |
|---|---|
| 终端收起后出现“游离拖动条” | `--term` 仍是 170px，而收起后的终端行只有 40px；左右手柄用 `bottom: var(--term)`、横向手柄用 `bottom: calc(var(--term) - 3px)`，位置比真实分界线高 130px |
| 终端和对话框横跨左右栏 | `.term-pane` 原为 `grid-column: 1 / span 3`；对话框被追加到最外层 `.app` 底部而非 `.main` 中栏 |
| 收起终端后仍有横向手柄占位 | 收起只改高度，没有把 `rzTerm` 从网格中移除 |
| 左右栏不能收起 | 侧栏宽度固定，只有拖拽没有折叠语义 |

## 3. 最终布局（5ef8fab）

`.main` 是唯一布局所有者：

```css
.main{display:grid;grid-template-columns:var(--left,264px) minmax(0,1fr) var(--right,360px);
      grid-template-rows:minmax(0,1fr) 8px var(--term,170px) auto}
.main.term-collapsed{grid-template-rows:minmax(0,1fr) 0px 40px auto}
```

- **左栏**：成员 / 模板，`x=0 w=264`，跨满全高到 800；顶部「成员」按钮收起/展开。
- **中栏**：画布 → 终端 → 底部对话框，`x=265 w=654`（1280 宽窗口）；对话框固定在中栏最底部。
- **右栏**：记录 / 私聊 / 窗口，`x=920 w=360`，跨满全高；顶部「记录」按钮收起/展开。
- **终端**：默认收起只留 40px 标题栏；展开为 170px（或上次拖拽高度）；收起时 `rzTerm` 为 `display:none`，宽高 0×0，不再悬空。
- **对话框**：仍为 2 行 `textarea` + 发送按钮，`composerBottom=800` 与窗口底部对齐，但宽度只到中栏（654px），不再压住左右栏。

## 4. 拖拽与收起语义

- 左右拖拽柄：拖动改 `--left` / `--right`，结果写 `localStorage`，双击复位。
- 拖到宽度 < 120px：**直接收起该侧栏**（`display:none`），按钮文案变「展开成员」/「展开记录」。
- 收起后边缘保留手柄，往外拖可恢复；实测左栏收起后按钮为「展开成员」，再点恢复 264px。
- 横向拖拽柄（`rzTerm`）：`pointerdown` 时自动展开终端再开始拖，避免“拖了但终端仍藏着”。
- 收起终端时把 `--term` 同步为 40px 并记住原高度，展开时恢复（默认 170px，或拖拽后的高度）。

## 5. 验证证据（真实浏览器 1280×800）

```
main   x=0    w=1280  b=800
left   x=0    w=264   b=800
right  x=920  w=360   b=800
stage  x=265  w=654   b=635
term   x=265  w=654   b=677   （收起时 h=40）
composer x=265 w=654  b=800
rzTerm x=265  w=654   y=498..506   termTop=507   （展开时）
rows: 453px 8px 170px 122px  /  595px 0px 40px 122px
```

- 收起：左栏 `display:none`、按钮「展开成员」；右栏同理；`rzTerm` `display:none/0×0`。
- 展开：`rzTerm` `display:block`，宽度 654 等于中栏，正好压在中栏分界线上。
- 拖动左栏手柄到 x=50：自动收起；控制台报错为空。

## 6. 可复用经验

- **布局变量必须只有一份真值**：网格行高/列宽（`grid-template-*`）与 CSS 变量（`--term` / `--left` / `--right`）要同步；只改高度不改变量会产生“浮空拖拽条”。
- **拖拽柄的位置应派生自实际分界**，而不是历史默认值；面板收起时应把对应手柄移出布局（`display:none`），不要只改尺寸。
- 三栏产品的默认收敛方式是“中间主区 + 可收起侧栏”：中间放主工作流（画布、终端、对话框），左右只放导航/记录；参考 DeepSeek Harness / OpenCode 的侧栏思维。
- 收起阈值（<120px 自动收起）比只给按钮更贴近市面编辑器；收起后必须保留恢复入口（边缘手柄 + 标题按钮）。
- 这类“游离元素”问题用 `getBoundingClientRect` 对比分界线与手柄坐标即可定位，不要靠截图观感。

## 7. 边界

- 本轮未改动后端；服务仍在 `http://127.0.0.1:7600`（启动时曾报 `EADDRINUSE`，说明已有实例在跑，健康检查 200）。
- 工作区出现过未跟踪目录 `data/edge-primary/`、`data/edge-stray/`（浏览器验证用的临时数据），提交中未纳入。
