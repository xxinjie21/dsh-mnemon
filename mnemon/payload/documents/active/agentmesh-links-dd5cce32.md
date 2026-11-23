---
id: "dd5cce32-7258-498e-8f52-ac5a27a867e3"
title: "AgentMesh 用户自定义通信线（links）：设计边界、实现落点与验证"
description: "用户自定义连线 links 的边界裁决（路径定义而非路由规则）、schema v14/API/UI 实现落点、验证证据，以及与一期定稿“无路由规则引擎”的关系。"
status: "active"
created_at: "2026-10-29T18:06:21.645Z"
updated_at: "2026-10-29T18:06:21.645Z"
content_hash: "f3d5ee7d9474282f87a2d4d0fdaf6f0e458f8a02c1c370c258bda787f5634fd6"
source_paths:
  - "src/core/links.js"
  - "src/core/db.js"
  - "src/api/http.js"
  - "ui/app.js"
  - "ui/styles.css"
  - "docs/links.md"
  - "test/unit/links.test.js"
session_ids:
  - "ba050124-3b6c-4ed5-8860-e086b5f462f4"
memory_body_ids:
  []
---

# AgentMesh 用户自定义通信线（links）与既有"无规则引擎"定稿的衔接

> 工作区 `D:\ZISHIKU\AI\AgentMesh`，提交 `e240464 feat(ui): user-defined communication links`（8 files, +193/−3）。
> 关联既有文档：d373d32b-0b73-4671-8066-ccf944a194b2（一期设计定稿，含"三条路由、不做用户可编程路由规则引擎"）。

## 1. 起点：用户问"通讯线现在能不能用户自定义"

定稿阶段画布上的线只是 **组中心连成员** 的装饰虚线，由分组自动生成，不可编辑、不落库、不参与路由。
用户提出要把线变成可自定义的实体，因此新增 `links` 概念，同时必须回答"这条线是否改变已发消息的收件人"。

## 2. 边界决定（本功能的核心裁决）

- 线 = **路径定义 + 可视化**，可新增、可删除、可持久化；
- 线**不**拦截未连线成员发消息，**不**改变已发消息的收件人，发送仍按用户所选目标路由；
- 因此与定稿中"不做用户可编程路由规则引擎"不冲突：links 不是路由规则，只是用户画出的通信路径；
- 若要"只允许沿线通信"的强制约束，需要另加一层路由权限（明确列为下一步，未在本次实现）。

界面由此并存两种线：组中心→成员的虚线（分组示意，自动生成）与用户连的实线（links 表，可删）。

## 3. 实现落点

| 层 | 内容 |
|---|---|
| 存储 | `src/core/db.js` 迁移 version 14：`links(link_id PK, from_node_id, to_node_id, label, created_at, UNIQUE(from,to))`，两端外键 ON DELETE CASCADE，索引 `idx_links_from` |
| 领域 | `src/core/links.js`（新建），配套 `test/unit/links.test.js` |
| API | `GET /api/links`、`POST /api/links {from_node_id,to_node_id}`、`DELETE /api/links/{link_id}`；写入后 `hub.broadcast({type:'links_changed'})`；`src/api/http.js` |
| UI | `ui/app.js`（连线模式：先点起点再点终点；状态 `state.links`/`state.linkFrom`；点线或「路径」面板可删除）、`ui/index.html`、`ui/styles.css` |
| 文档 | `docs/links.md`（随本次提交新增） |

## 4. 验证证据

- 全量测试：119 pass / 0 fail / 2 skip（`migrate(db) === LATEST_SCHEMA_VERSION` 等断言随 schema 14 通过）；
- API 实测：create 200 → 列表 1 条 → delete 200 → 归零后再查为 0；
- UI 实测（真实浏览器）：`state.links=1`，画布 `.edge-svg path` 计 1，管理面板显示 "A → B / 删除"，按钮文案变为「取消连线」；刷新后保留（已入库）；
- 服务 `http://127.0.0.1:7600`，schema_version=14。

## 5. 遗留

- 强制路由（沿线才允许通信）未做；
- 连线目前无方向语义标注（label 字段预留，UI 未编辑）。

## 6. 同批次的另一个调查：输入框文字/光标不可见

用户报"输入框文字看不见，光标都没有"。根因不是逻辑问题：`textarea` 只设了背景、字体，未设 `color`，深色底上仍用浏览器默认黑字；且旧 CSS 被缓存。
修复沿用 `ui/styles.css` 中 `.composer-box .d-in` 的显式 `color`/`caret-color`/`-webkit-text-fill-color` 与 `color-scheme: dark`，并给静态 UI 文件加 `Cache-Control: no-store` 以避免继续吃旧缓存（提交 `a2a155c`、`b4ba46a`；`src/api/http.js` 静态文件响应头）。教训：深色主题下表单控件必须显式声明文字色，改动样式后用真实浏览器截图 + `getComputedStyle` 验证，并考虑缓存。

## 7. 关键路径

`src/core/db.js`、`src/core/links.js`、`src/api/http.js`、`ui/app.js`、`ui/index.html`、`ui/styles.css`、`docs/links.md`、`test/unit/links.test.js`。
