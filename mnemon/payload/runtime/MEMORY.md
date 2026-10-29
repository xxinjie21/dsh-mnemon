DSH skills-manager (dsh-skills-manager 插件) 的 skill 启用状态存于 ~/.dsh/skills-manager/state.json；disabledSkills[rootKey] 在插件加载时过滤 skill（rootKey `agents` 对应 DSH_AGENTS_HOME || ~/.agents），skills 目录在会话启动时装配，改动后需重载 DSH 才对新会话生效。UI 与直接编辑该文件是同一份状态。
§
本项目（tcm-ehr-qc）约定：改完代码必须自行验证；改动只提交，不推送远端，推送需用户明确指示。修改 Java 源码后需 mvn clean 再测（ops_edit_file 保留原 mtime，会致 Maven 增量编译跳过已改文件）。
§
用户不接受在已居中的 flex 按钮区内追加控件（会让按钮被顶偏）；Governance.vue 清洗按钮区的开关等新控件要放独立一行，原按钮区保持原样。
§
tcm-ehr-qc 的 AI 智能解读三个功能（质控解读 AiInterpretCard / 术语补词建议 termsuggest / AI 预检建议 runAiAsync('review')）均已确认保留、不删：三者都有真实调用方，且 LLM 未启用时有降级（显示「规则预检」），演示不会翻车。曾误判 aiReview() 为死代码而删掉前端同步封装——实际该功能走异步路 runAiAsync，/ai/review 端点与 AiService.review() 未受影响。
§
AgentMesh 测试口径：按里程碑统一跑——每个 M 只跑一次该 M 全部 task 的测试（npm run test:mN），不按单个 task 分别跑；真机 e2e 另跑并记入证据文档。
