---
id: "657ebab7-c6e8-46eb-a5e8-72c38c2890a0"
title: "AgentMesh 原型前端接入后端：路由契约、异常矩阵与桥接映射"
description: "AgentMesh 原型（/）接后端阶段：路由白名单与 /classic 归档、404/5xx/网络异常文案与按钮边界、prototype-bridge.js 的能力映射、未接入清单、品牌与可见性教训、提交链。"
status: "active"
created_at: "2026-10-30T05:09:36.278Z"
updated_at: "2026-10-30T05:09:36.278Z"
content_hash: "8f874fabe588bceb996eccb066fe8f64ee09701f9eff926e04ab1b16d54b12a2"
source_paths:
  - "src/api/http.js"
  - "src/api/page-errors.js"
  - "ui/route-guard.js"
  - "prototype/prototype-bridge.js"
  - "prototype/AgentMesh-原型.html"
  - "docs/frontend/agentmesh-frontend-contract.md"
  - "test/unit/frontend-pages.test.js"
session_ids:
  - "ecc3b5bc-d348-4518-9586-8c5c26924997"
memory_body_ids:
  []
---

# AgentMesh 原型前端接入后端：路由契约、异常矩阵与桥接映射

> 工作区 `D:\ZISHIKU\AI\AgentMesh`。2026-10-30 同一会话的第三阶段：先做路由守卫与异常兜底，再改品牌，最后把原型接到真实后端。前两阶段边界见文档 `672f0d19-ab57-44fc-892f-4224d5dfb5b9`（范围收敛 + 改本体/禁止桥接副本）；本文只记录本阶段新增的路由契约、异常策略、桥接映射与未接入清单，不复述其内容。

## 1. 阶段演进与规则覆写

1. 用户先要求「只改 `prototype/AgentMesh-原型.html` 本体、当前先不接后端」，据此删掉副本与桥接（提交 `160cd20`）。
2. 随后用户明确「现在将 `http://127.0.0.1:7600/` 接入后端」——**桥接禁令被显式指令取代**，桥接层恢复，但形态改为在原型目录内（`prototype/prototype-bridge.js`），不再复制原型文件。
3. 仍长期有效的约束：**改原型文件本体，不复制页面、不换版式、不做大规模重构**；新能力只做增量入口。

## 2. 路由契约（`src/api/http.js`）

- `/`、`/prototype`、`/prototype.html` → `prototype/AgentMesh-原型.html` 原文件（主界面，后续全部迭代在此）。
- `/classic`、`/index.html` → `ui/index.html`（旧界面，仅归档对照，不新增功能）。
- `/route-guard.js`、`/prototype-bridge.js` 作为静态脚本单独出；`/vendor/xterm*` 直接从依赖出，避免二次打包。
- 未知路径不冒充页面，仍按静态资源处理；路径用白名单映射表（`PROTO_FILES`/`UI_FILES`），不做目录遍历。
- 文件缺失 → 404 错误页模板；读取异常 → 500 错误页；模板集中在 `src/api/page-errors.js`。

## 3. 异常状态矩阵（服务端 + 客户端双保险）

| 场景 | 页面 | 文案 | 按钮 | 自动跳转 |
|---|---|---|---|---|
| 2xx | 任意 | 无 | 无 | 否 |
| 404 | `/` | 原始前端原型页面资源未找到，可访问旧版保留界面 | 访问旧版归档界面 → `/classic` | 否 |
| 404 | `/classic` | 旧版归档界面资源不存在 | 返回新版原型 → `/` | 否 |
| 5xx | `/` | 页面服务内部异常 | 可选「访问旧版归档界面」 | 否 |
| 5xx | `/classic` | 页面服务内部异常 | 无（不反跳新版） | 否 |
| 网络失败/超时 | 两页 | 前端服务连接失败，请确认服务是否正常启动 | 仅「关闭」 | 否 |

- 客户端守卫：`ui/route-guard.js`，两页分别挂 `data-page="new"` / `data-page="classic"`；只走页面内提示，不用原生 `alert/confirm`。
- 硬约束：错误文案面向业务用户，**不得输出堆栈、URL、端口、文件路径**；按钮只是备选入口，不强制跳转；不编造不存在的接口地址。
- 校验方式：`test/unit/frontend-pages.test.js` 断言文案/按钮/无技术信息泄露；真实浏览器实测网络错误无跳转、新版 5xx 有 `/classic`、旧版 5xx 无 `/`。

## 4. 桥接映射（`prototype/prototype-bridge.js`，提交 `1f4754f`）

桥接层只做「数据映射 + 操作转译」，不碰原型布局与视觉：

| 原型位置 | 已接后端能力 |
|---|---|
| 顶栏「连接本机」 | 申请 token、注册本机 GUI 节点（`ui-local`） |
| 左侧任务线/会话 | 任务列表、状态、负责人 |
| 顶部 Agent 标签 | 任务相关成员 |
| 中间聊天 | 任务消息（发送者/接收者/时间） |
| 底部 Composer（Enter） | 真实发送消息 |
| 左侧「+」 | 创建任务并派给成员 |
| 暂停/继续 | 当前负责人暂停/恢复 |
| 画布 | 真实成员/任务概览 |
| 右侧泳道 | 由真实消息派生的任务事件 |

未接入、不得假装可用：接入成员/模板向导、通知、任务看板完整操作（取消/归档/重开）、自定义通信线与通信流、系统状态/诊断、嵌入式终端、消息撤回/重试。

## 5. 品牌与可见性经验

- 原型自带 `Lancer` 品牌（`<title>`、顶栏 logo、示例数据）。提交 `3ff6976` 只改品牌层：标题 `AgentMesh · 多 Agent 工作台（前端原型）`、logo `AM`、品牌 `AgentMesh`、加「前端原型 · 静态数据」标识；**示例任务线/用户数据里的 `Lancer` 保留**，不当作品牌一并替换。
- 教训：路由守卫只在 404/5xx/网络错误分支出现，正常浏览不可见。用户反馈「没看到改动」时，先把「只在错误路径生效」讲清并补上首屏可见变化，不要把守卫算作页面改动。

## 6. 验证与提交

- 全量：`124 pass / 0 fail / 2 skip`；`/`、`/classic`、`/route-guard.js` 均 200。
- 真机验证靠无头浏览器取 DOM 快照（品牌、badge、guard、聊天文本、toast），不靠肉眼声明。
- 提交链：`160cd20`（撤桥接、/ 指原型本体）→ `3a9b26b`（路由守卫 + 页面级错误态 + `docs/frontend/agentmesh-frontend-contract.md`）→ `3ff6976`（品牌改 AgentMesh）→ `1f4754f`（原型接后端）。
- 契约文档 `docs/frontend/agentmesh-frontend-contract.md` 记录组件清单、mock 字段（agent/thread/chain/message/step/task/notice）、状态枚举 `ready/running/waiting/done/failed/closed`、消息模式 `send/request/reply` 与前后端对齐检查项。
