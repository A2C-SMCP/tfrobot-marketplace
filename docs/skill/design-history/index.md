# SKILL 协议设计史（A1\~A4）

下列文档是 [SKILL 协议 v1](../protocol-v1.md) 的分步设计稿，按 Jira 子任务顺序产出。

**冲突仲裁**：如与协议 v1 表述冲突，**以协议 v1 为准**。本目录文档保留作为决策 rationale 与设计史，不作为实施契约。

对应 Story：[TFRS-180](https://turingfocus.atlassian.net/browse/TFRS-180)；Epic：[TFRS-179](https://turingfocus.atlassian.net/browse/TFRS-179)。

| 子任务 | Jira | 文档 | 核心结论 |
| --- | --- | --- | --- |
| A1 | [TFRS-183](https://turingfocus.atlassian.net/browse/TFRS-183) | [frontmatter 字段表](frontmatter-fields.md) | 对齐 Agent Skills 开放标准 + 1 独有字段 `runtime`；不采纳 Claude Code 扩展字段 |
| A2 | [TFRS-184](https://turingfocus.atlassian.net/browse/TFRS-184) | [.skillenv 设计](skillenv.md) | 标准 dotenv 双语义；ManagedLLM keystore 三层强制隔离 |
| A3 | [TFRS-185](https://turingfocus.atlassian.net/browse/TFRS-185) | [runtime 枚举](runtime-enum.md) | 双引擎并存（CubeSandbox / E2B）+ 引擎前缀语法 + BYO Dockerfile |
| A4 | [TFRS-186](https://turingfocus.atlassian.net/browse/TFRS-186) | [目录与占位符](directory-placeholders.md) | `TFROBOT_*` 私有命名空间 + 纯函数沙箱模型 + `skills` 工具契约 |
