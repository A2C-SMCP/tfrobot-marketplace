# SKILL 协议（TFRobotServer v1）

TFRobotServer 的 SKILL 协议参考 [Anthropic Agent Skills 开放标准](https://agentskills.io/specification) 与 [Claude Code Skills 文档](https://code.claude.com/docs/en/skills) 设计，并针对多 Pod K8s 部署、Sandbox 沙箱执行（CubeSandbox 主 / E2B 备）、密钥托管等差异进行受控扩展。

对应 Epic：[TFRS-179](https://turingfocus.atlassian.net/browse/TFRS-179)。

## 权威规范

| 文档 | 范围 |
| --- | --- |
| **[SKILL 协议 v1](protocol-v1.md)** ⭐ | SKILL 内容契约（frontmatter / 目录 / runtime / 占位符 / `skills` 工具 / 安全模型）—— Module A/B/C 实施基准 |

新读者从这份开始。**SKILL 包的分发约定**（Git 仓库 / `marketplace.json` / `plugin.json`）属于姊妹章节 [Marketplace 分发](../marketplace/index.md)。

## 模块对应

| 模块 | Story | 范围 | 规范文档 |
| --- | --- | --- | --- |
| A — 文档格式协议 | [TFRS-180](https://turingfocus.atlassian.net/browse/TFRS-180) | SKILL.md 内容约定 | [SKILL 协议 v1](protocol-v1.md)（含 A1\~A4 设计史） |
| B — 存储与同步 | [TFRS-181](https://turingfocus.atlassian.net/browse/TFRS-181) | Git ↔ MinIO ↔ Postgres / `skills` 工具 / 资源访问安全 | [SKILL 协议 v1 §10, §13.1](protocol-v1.md) |
| C — 执行引擎 | [TFRS-182](https://turingfocus.atlassian.net/browse/TFRS-182) | Sandbox 双集群 / 模板 / 注入链路 | [SKILL 协议 v1 §6, §9, §13.2](protocol-v1.md) |
| D — Marketplace 分发 | [TFRS-201](https://turingfocus.atlassian.net/browse/TFRS-201) | Git 仓库分发标准（兼容 Claude Code）+ 多 plugin 多 SKILL 包装 | [Marketplace v1 规范](../marketplace/protocol-v1.md) |

## 子任务产出（设计史）

A1\~A4 是模块 A 协议规范的分步设计文档；如与 SKILL 协议 v1 表述冲突，**以协议 v1 为准**。保留作为决策 rationale 与设计史，统一收录于 [设计史索引](design-history/index.md)。

| 子任务 | Jira | 状态 | 产出 |
| --- | --- | --- | --- |
| A1 frontmatter 字段表 | [TFRS-183](https://turingfocus.atlassian.net/browse/TFRS-183) | 已完成 | [frontmatter 字段表](design-history/frontmatter-fields.md) |
| A2 `.skillenv` 环境变量声明 | [TFRS-184](https://turingfocus.atlassian.net/browse/TFRS-184) | 已完成 | [.skillenv 设计](design-history/skillenv.md) |
| A3 `runtime` 枚举（双引擎并存 / 引擎前缀语法 / BYO Dockerfile） | [TFRS-185](https://turingfocus.atlassian.net/browse/TFRS-185) | 已完成 | [runtime 枚举](design-history/runtime-enum.md) |
| A4 目录结构与占位符（`TFROBOT_*` 私有命名空间，3 个占位符 + `skills` 工具） | [TFRS-186](https://turingfocus.atlassian.net/browse/TFRS-186) | 已完成 | [目录与占位符](design-history/directory-placeholders.md) |
| A5 v1 规范整合 | [TFRS-187](https://turingfocus.atlassian.net/browse/TFRS-187) | 进行中 | [SKILL 协议 v1](protocol-v1.md) |
