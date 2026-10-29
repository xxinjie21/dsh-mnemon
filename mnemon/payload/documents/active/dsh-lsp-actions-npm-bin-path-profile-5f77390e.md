---
id: "5f77390e-3add-45eb-9bbc-11ee6d5549d1"
title: "dsh-lsp-actions 落地：插件安装后配置语言服务器、npm 全局 bin 不在 PATH 的根因与 profile 覆盖写法"
description: "dsh-lsp-actions v0.5.15 已装但默认 servers 为空导致不可用；补齐 typescript-language-server、修复 D:\\nodejs-global 缺失 PATH、在 profile cordis.patch.yml 追加 servers 覆盖的完整证据、配置片段与生效前提。"
status: "active"
created_at: "2026-10-19T16:59:52.618Z"
updated_at: "2026-10-19T16:59:52.618Z"
content_hash: "aa24e6852b2a0e7fcc78db8c69f7449b6dad294ba63bd6be290f1bd1db492222"
source_paths:
  []
session_ids:
  - "20aff59b-8974-44bb-9a19-7b35289cdf8f"
memory_body_ids:
  []
---

# dsh-lsp-actions 落地记录（2026-10-20）

背景：用户目标是全栈开发。经插件市场盘点，17 个已装插件覆盖较广，但**代码智能（LSP）是结构性缺口**——此前只能靠全文搜索和读文件，无法查类型定义、找引用、拿真实类型错误。用户据此安装 `dsh-lsp-actions`。

## 结论先行

插件装上 ≠ 可用。`dsh-lsp-actions` 的默认配置是 `servers: {}`（**默认不激活任何语言服务器**），而本机当时**一个语言服务器都没装**。因此必须完成三件事才算落地：

1. 安装语言服务器二进制（本次：`typescript-language-server` 6.0.2 + `typescript` 7.0.2）
2. 让 DSH 进程能在 PATH 上解析到它（修 PATH 或写绝对路径）
3. 在 profile 层覆盖 `servers` 表

## 环境事实（重要，非本项目专属）

`npm prefix` = `D:\nodejs-global`，全局 bin 目录里已有 `agent-browser`、`claude`、`tsc`、`pnpm`、`typescript-language-server` 等。**但该目录不在任何持久 PATH 中**（用户级与机器级 PATH 都没有它；PATH 里只有 `D:\nodejs-v24`）。

后果：`npm i -g` 装上的东西全部无法按命令名调用。本次已把 `D:\nodejs-global` 追加进用户级 PATH（位于 `D:\nodejs-v24` 之前），原值备份在 `%TEMP%\wbx-076\user-path.bak`。

> 排查提示：`Get-Command <name>` 找不到但文件确实在 `D:\nodejs-global` 时，先怀疑 PATH，不要怀疑安装失败。

## 插件与配置位置

- 插件包：`C:\Users\xxinj\.dsh\profiles\desktop\node_modules\dsh-lsp-actions`，v0.5.15，已在 profile 的 `bundles` 与 `dependencies` 中登记。
- 插件自带 patch：同目录 `cordis.patch.yml`，注入 id `lsp-actions`，`inject: [tools, fs, subprocess]`。
- 用户可覆盖的 profile 层：`C:\Users\xxinj\.dsh\profiles\desktop\cordis.patch.yml`（修改前备份为 `cordis.patch.yml.bak-20261020-005828`，sha256 前值 `6309AF3F...46DCA8`）。

注意：以上路径均在工作区（`C:\Users\xxinj\Desktop`）之外，因此未登记为文档 sourcePaths。

## 追加到 profile 末尾的条目

```yaml
- id: lsp-actions
  name: dsh-lsp-actions
  config:
    servers:
      typescript:
        command: typescript-language-server
        args: [--stdio]
        extensionToLanguage: {.ts: typescript, .tsx: typescriptreact,
                              .js: javascript, .jsx: javascriptreact,
                              .mjs: javascript, .cjs: javascript}
        projectMarkers: [package.json, tsconfig.json]
```

字段语义（来自插件 README-zh）：`command` 与 `extensionToLanguage` 必填，`command` 在**加载期**于 PATH 上解析；`projectMarkers` 让同名扩展名按最近的工程配置文件路由——monorepo 里不同子项目各用各的 tsconfig，无需写路径规则。

路由优先级：`fileGlobs` 命中 → 向上找到最近的 `projectMarkers` 目录且该条目映射此扩展名 → 按配置顺序取第一个映射该扩展名的条目。

## 有意未改的项

`editor.enabled` 保持插件默认 `false`。插件注释说明：Web/CLI profile 自己管理 stdio，该后端只在专门的 headless 后端组合（如 `examples/vscode/backend`）里才开。不是那种组合就不应改。

## 验证与生效前提

已验证：YAML 可解析、顶层 17 条、`lsp-actions` 正确、`servers: typescript` 正确、其余 16 条逐条未动；`typescript-language-server --version` 实际执行返回 6.0.2，`tsc --version` 返回 7.0.2（不是只看文件名存在）。

**必须重启 DSH 才生效**：插件配置在启动时读取，PATH 变更也需新进程继承。重启后 `lsp_diagnostics` / `lsp_rename` / `lsp_code_actions` 才会注册。

若重启后仍报 command not found，说明 DSH 进程没继承新 PATH，此时把 `command` 改成绝对路径 `D:\nodejs-global\typescript-language-server.cmd`。

## 记过的坑

- PowerShell here-string 里的变量插值不会生效（写验证脚本时会拿到 `undefined` 路径并抛 `ERR_INVALID_ARG_TYPE`）；改用参数传递或直接写文件。
- 复用临时脚本前要先读文件，否则会被"file has not been read"拦下。

## 相关文档

- `bf2bcf5d-39e1-4c5d-b005-0d8452bb1149`：同一 profile 文件 `cordis.patch.yml` 中 mnemon 块的最小化配置规范，可作为本文件条目写法的一致性参照。
- `7bec5266-1740-4f7d-a252-7ad7a78094e4`：同一环境里的 skill 与插件来源冲突治理（.agents 与 aegis）。
