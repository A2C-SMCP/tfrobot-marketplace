# Marketplace 分发（TFRobot）

TFRobot Marketplace 定义 **Plugin 包**（含 SKILL 与 MCP Server 配置）从 Git 仓库到平台的分发流程，与 [Claude Code Plugin Marketplaces](https://code.claude.com/docs/en/plugin-marketplaces) 兼容并按 TFRobot 多租户、Curator 审核、A2C-SMCP MCP 配置等需求做受控扩展。

对应 Story：[TFRS-201](https://turingfocus.atlassian.net/browse/TFRS-201)；Epic：[TFRS-179](https://turingfocus.atlassian.net/browse/TFRS-179)。

## Plugin 内容

一个 Plugin 由两类内容组成（至少含一项非空）：

* **SKILL 子树**（`<plugin>/skills/`）—— 单个 SKILL 包的内容契约见姊妹章节 [SKILL 协议](../skill/index.md)
* **MCP Server 子树**（`<plugin>/mcp-servers/`）—— 配置 schema 完整对齐 [A2C-SMCP v0.3.2](https://doc.turingfocus.cn/a2c-smcp/)（本规范 §8 摘要）

## 权威规范

| 文档 | 范围 |
| --- | --- |
| **[Marketplace 规范](protocol.md)** ⭐ | Git 仓库分发标准 —— `marketplace.json` / `plugin.json` JSON Schema、`source` 类型枚举、目录布局（SKILL + MCP）、Curator 模式 |

## 实施参考（运行时行为，非规范契约）

| 文档 | 范围 |
| --- | --- |
| [加载行为参考](loading-behavior.md) | Plugin manifest × strict mode 组合矩阵、缺失兜底、冲突检测 —— 基于 Claude Code 源码观察，给 Module D 实施工程师与 SKILL 作者参考 |
