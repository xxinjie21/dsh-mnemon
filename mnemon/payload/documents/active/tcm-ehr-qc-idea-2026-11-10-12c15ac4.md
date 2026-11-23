---
id: "12c15ac4-5726-4edd-b318-fd795c3b0f63"
title: "tcm-ehr-qc 本地启动与 IDEA 运行环境排障手册（2026-11-10）"
description: "IDEA 报 ClassNotFoundException 的根因证据（空白 iml + output=out/ + 未导入 Maven）、IDEA 修复步骤、命令行启动后端脚本与 Windows/MSYS 踩坑、四端口运行态与冒烟验证基线。"
status: "active"
created_at: "2026-11-10T10:31:48.801Z"
updated_at: "2026-11-10T10:31:48.801Z"
content_hash: "a1d1fa0d3f76fe684d117e315e60f4275cf3a7277f16e9aad8b355146c16c702"
source_paths:
  - ".idea/misc.xml"
  - ".idea/tcm-ehr-qc.iml"
  - "java-backend/pom.xml"
session_ids:
  - "mnemon-run-v1-b27bf110-45c7-46b2-8b5b-d73b4318df10"
memory_body_ids:
  []
---

# tcm-ehr-qc 本地启动与 IDEA 运行环境排障手册（2026-11-10）

工作区 `D:\ZISHIKU\AI\tcm-ehr-qc`。本文记录一次「IDEA 跑主类报 ClassNotFoundException + 前后端未启动」的排查结论，以及本机可复用的启动/验证步骤。关联文档：`eaa00c66-9d76-425b-b317-8e36f63fd1fd`（完整验收评审，其中 P0-1 指出缺部署文档）、`9e437106-8ac7-45f4-89de-6938a08302e7`（主线与非主线范围）。

## 1. 结论先行

- `ClassNotFoundException: com.tcm.ehr.TcmEhrApplication` **不是编译问题、也不是代码问题**，而是 IDEA 侧 classpath 没接上 Maven 产物。
- 后端用 Maven 启动即可正常提供服务；前端 3000 已在跑时无需重启。

## 2. 根因证据（IDEA 侧）

| 证据 | 现状 | 含义 |
|---|---|---|
| `.idea/misc.xml` | `<output url="file://$PROJECT_DIR$/out" />`，`project-jdk-name="24"` | IDEA 输出到仓库根 `out/`，而 Maven 产物在 `java-backend/target/classes` |
| `.idea/tcm-ehr-qc.iml` | `<module type="JAVA_MODULE">` + `<content url="file://$MODULE_DIR$" />` + `<orderEntry type="inheritedJdk" />` | 整个项目只有一个**空白 Java module**，没有任何 Maven 依赖 |
| `java-backend/*.iml` | 不存在 | IDEA 从未把 `java-backend/pom.xml` 当作 Maven 项目导入 |
| `.idea/compiler.xml` | 不存在 | 无 Maven 模块的编译输出映射 |
| `java-backend/target/classes/.../TcmEhrApplication.class` | 存在；共 340 个 class，字节码 major=61（JDK17） | 编译产物完整、版本与 `pom.xml` 的 `java.version=17` 一致 |

因此 IDEA 直接点绿色 Run 时，classpath 指向空的 `out/` 且缺 Spring 依赖 jar → 必然 ClassNotFound。

## 3. IDEA 修复步骤（需先关闭 IDEA 再改 `.idea`，否则会被覆盖）

1. 项目树右键 `java-backend/pom.xml` → **Add as Maven Project**（或 Maven 面板 `+` 选中该 pom）。
2. File → Project Structure → Modules：确认出现 `java-backend` 模块，输出目录为 `target/classes`。
3. 建议 Project SDK 从 **24 改成 17**（项目 `java.version=17`，IDEA 默认 JDK24 编译易踩语言级别不一致）。

## 4. 命令行启动后端（已验证可用）

前置：`JAVA_HOME=F:\jdk17`，Maven 在 `D:\devSoft\apache-maven-3.9.16\bin\mvn.cmd`，`java-backend` 下执行离线模式 `mvn -o spring-boot:run`。

现成脚本：`C:\Users\xxinj\start-backend-tcm.bat`

```bat
@echo off
set JAVA_HOME=F:\jdk17
cd /d D:\ZISHIKU\AI\tcm-ehr-qc\java-backend
call D:\devSoft\apache-maven-3.9.16\bin\mvn.cmd -o spring-boot:run
```

**Windows/MSYS 踩坑记录（复用价值高）**：
- 在 bash 里传 `JAVA_HOME=/f/jdk17`（MSYS 风格）给 `mvn.cmd` 无效，批处理不认 — 必须 Windows 风格 `F:\jdk17`。
- `cmd //c "...&&..."` 在 bash 下参数会被吞掉，只进交互式 shell 不执行 — 改为**落一个 .bat 文件**再执行，或用 PowerShell 调用脚本。
- 该会话中 `pwsh` 工具不可用（`tools.pwsh is not a function`），PowerShell 只能经 bash 间接调用。

## 5. 运行态与验证基线

| 服务 | 端口 | 备注 |
|---|---|---|
| MySQL | 3306 | |
| 后端 API | 8080 | 启动日志：ES 连接正常、数据初始化完成、报告预热约 6977ms |
| Elasticsearch | 9200 | 7.17.x |
| 前端 dev | 3000 | Vite，代理 3000→8080 |
| NLP 服务 | 8001 | 需单独 `uvicorn main:app --host 127.0.0.1 --port 8001`（python-nlp/.venv），未启动时抽取降级 |

冒烟验证（全部应 200）：登录 `POST /api/auth/login`、首页看板、标准化报告、复核任务、清洗任务、我的组织；前端首页与代理路径各 200；`mvn test` 632 用例全通过、`vite build` 约 8.5s。

## 6. 复用提醒

- 验收/演示前先确认 3306/8080/9200/3000 四个端口在线，8001 决定「结构化解析」是否降级。
- `mvn clean` 会清空 `target`，之后再从 IDEA Run 会重现 ClassNotFound；命令行启动不受影响。
