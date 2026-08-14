# SKILL 协议（TFRobot）

TFRobot 的 SKILL 协议参考 [Anthropic Agent Skills 开放标准](https://agentskills.io/specification) 与 [Claude Code Skills 文档](https://code.claude.com/docs/en/skills) 设计，聚焦 **SKILL 内容契约 + LLM 与 SKILL 的交互**。SKILL 的执行后端（沙箱 / 引擎 / 容器 / 凭证基础设施）归 Computer 侧 [A2C-SMCP](https://github.com/A2C-SMCP) 协议表达，本协议不规定。

对应 Epic：[TFRS-179](https://turingfocus.atlassian.net/browse/TFRS-179)。

## 权威规范

| 文档 | 范围 |
| --- | --- |
| **[SKILL 协议](protocol.md)** ⭐ | SKILL 内容契约（frontmatter / 目录 / `.skillenv` / 占位符 / `skills` 工具 / 安全模型）+ Robot 多来源汇聚视角 |

新读者从这份开始。**SKILL 包的 Git 仓库分发约定**（`marketplace.json` / `plugin.json`）属于姊妹章节 [Marketplace 分发](../marketplace/index.md)；**SKILL 在 Computer 侧的执行后端**则归 A2C-SMCP 实施方文档。

## SKILL 归属与来源（一句话视角）

SKILL 归属于 **Robot**，切换 Robot 即切换 SKILL 集合。Robot 的 SKILL 集合由多来源汇聚：

* **云端 Robot 配置**——从云端对象存储的约定目录拉取（默认根 `skills/`，TFRobot 平台采用 MinIO 实现）。仅文档驱动，不执行脚本。
* **已连接的 Computer**——通过 [A2C-SMCP](https://github.com/A2C-SMCP) 暴露本地可用 SKILL，可执行 `scripts/`。
* **Marketplace 仓库**——Git 仓库分发，详见 [Marketplace 规范](../marketplace/protocol.md)。

不同来源带来不同的执行能力，但**落到加载方手中的都是等价的 SKILL 文件夹**——内容契约完全一致。

## 子任务产出（设计史）

A1\~A4 是协议规范的分步设计文档；如与协议表述冲突，**以协议为准**。保留作为决策 rationale 与设计史，统一收录于 [设计史索引](design-history/index.md)。

| 子任务 | Jira | 状态 | 产出 |
| --- | --- | --- | --- |
| A1 frontmatter 字段表 | [TFRS-183](https://turingfocus.atlassian.net/browse/TFRS-183) | 已完成 | [frontmatter 字段表](design-history/frontmatter-fields.md) |
| A2 `.skillenv` 环境变量声明 | [TFRS-184](https://turingfocus.atlassian.net/browse/TFRS-184) | 已完成 | [.skillenv 设计](design-history/skillenv.md) |
| A3 `runtime` 枚举 | [TFRS-185](https://turingfocus.atlassian.net/browse/TFRS-185) | 已完成（撤回） | [runtime 枚举](design-history/runtime-enum.md) |
| A4 目录结构与占位符 | [TFRS-186](https://turingfocus.atlassian.net/browse/TFRS-186) | 已完成 | [目录与占位符](design-history/directory-placeholders.md) |
| A5 规范整合 | [TFRS-187](https://turingfocus.atlassian.net/browse/TFRS-187) | 进行中 | [SKILL 协议](protocol.md) |
