# MCP Server 配置（TFRobot）

TFRobot Plugin 可以在 `<plugin>/mcp-servers/` 子树下声明若干 **MCP Server 配置**，由 Robot 在加载 Plugin 时按 [A2C-SMCP 协议 v0.3.2](https://doc.turingfocus.cn/a2c-smcp/) 装配并通过 Computer 侧暴露给 LLM 工具循环。

本协议**不另行设计** MCP 配置 schema：完整 schema 来自 A2C-SMCP；本协议**只规定** Plugin 内的目录约定 + 摘要当前对齐版本（v0.3.2）的字段速查，便于 Plugin 作者撰写时无需切换文档。

> 动手前先看：[配置 MCP Server](../guides/configure-mcp-server.md)（场景化流程）；本章是字段级契约。

## 权威规范

| 文档 | 范围 |
| --- | --- |
| **[MCP Server 配置规范](protocol.md)** ⭐ | Plugin 内 `mcp-servers/` 目录约定 + A2C-SMCP v0.3.2 schema 摘要（BaseMCPServerConfig / stdio·streamable·sse / inputs / ToolMeta） |

## 与上游标准的关系

* **Schema 权威**：[A2C-SMCP § MCP Server 配置结构](https://doc.turingfocus.cn/a2c-smcp/latest/specification/data-structures/#mcp-server-配置结构)
* **当前对齐版本**：A2C-SMCP **v0.3.2**
* **同步策略**：随 A2C-SMCP **协议版本**同步（以协议为权威）；SDK 补丁版本（bugfix 末位）不构成同步触发
* **不重复设计**：本规范**不**为 MCP 配置发明替代字段、替代命名、替代占位符语法

## 与姊妹章节的关系

* [SKILL 协议](../skill/index.md) —— Plugin 的另一类内容（文档驱动的能力描述）；同一 Plugin 可同时含 SKILL 与 MCP Server
* [Marketplace 分发](../marketplace/index.md) —— Plugin 的 Git 仓库分发结构与 manifest schema
