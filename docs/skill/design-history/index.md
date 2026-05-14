# SKILL 协议设计史（A1\~A4）

下列文档是 [SKILL 协议 v1](../protocol-v1.md) 的分步设计稿，按 Jira 子任务顺序产出。

**冲突仲裁**：如与协议 v1 表述冲突，**以协议 v1 为准**。本目录文档保留作为决策 rationale 与设计史，不作为实施契约。

对应 Story：[TFRS-180](https://turingfocus.atlassian.net/browse/TFRS-180)；Epic：[TFRS-179](https://turingfocus.atlassian.net/browse/TFRS-179)。

| 子任务 | Jira | 文档 | v1 最终采纳 |
| --- | --- | --- | --- |
| A1 | [TFRS-183](https://turingfocus.atlassian.net/browse/TFRS-183) | [frontmatter 字段表](frontmatter-fields.md) | ✅ 完全对齐 Agent Skills 开放标准；不采纳 Claude Code 扩展字段 |
| A2 | [TFRS-184](https://turingfocus.atlassian.net/browse/TFRS-184) | [.skillenv 设计](skillenv.md) | ✅ 标准 dotenv 双语义；用户 vault 三层强制隔离 |
| A3 | [TFRS-185](https://turingfocus.atlassian.net/browse/TFRS-185) | [runtime 枚举](runtime-enum.md) | ❌ **v1 撤回**——A3 曾设计双引擎枚举 + BYO Dockerfile；v1 最终决定执行后端归 Computer 侧 A2C-SMCP 表达，`runtime` 字段不入协议。详见 [协议 v1 附录 A](../protocol-v1.md#附录-a设计史)。 |
| A4 | [TFRS-186](https://turingfocus.atlassian.net/browse/TFRS-186) | [目录与占位符](directory-placeholders.md) | ✅ `TFROBOT_*` 私有命名空间 + `skills` 工具契约 + 3 条安全原则 |
