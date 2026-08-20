# TFRobot Marketplace

**TFRobot Marketplace** 是 TFRobot 生态的市场平台，承载第三方扩展、Skill、Computer 配置等内容的发布、发现与分发流程。本站是它的**开放接口与配置规范文档**。

## 我要做什么？

按目标进入对应指南——每份指南走完「选型 → 配置 → 校验」全流程：

<div class="grid cards" markdown>

-   :material-file-document-edit:{ .lg .middle } __我要编写一个 SKILL__

    ---

    命名、frontmatter、目录布局、占位符、安全边界。

    [:octicons-arrow-right-24: 开始编写](guides/write-a-skill.md)

-   :material-server-network:{ .lg .middle } __我要配置 MCP Server__

    ---

    传输模式、配置字段、输入定义、占位符。

    [:octicons-arrow-right-24: 开始配置](guides/configure-mcp-server.md)

-   :material-store:{ .lg .middle } __我要发布 Plugin__

    ---

    仓库布局、marketplace / plugin manifest、安装与更新。

    [:octicons-arrow-right-24: 开始发布](guides/publish-plugin.md)

-   :material-sync:{ .lg .middle } __我要同时适配三个 Agent 系统__

    ---

    Claude Code / Codex / TFRobot 差异对照与技术选型。

    [:octicons-arrow-right-24: 开始适配](guides/three-platform-adaptation.md)

</div>

## 三大协议支柱

需要查字段约束 / schema 细节时进入规范页：

<div class="grid cards" markdown>

-   :material-puzzle:{ .lg .middle } __SKILL 协议__

    ---

    单个 SKILL 包的**内容契约**（frontmatter 7 字段含 `tags` 分类元数据、目录布局、`.skillenv`、占位符、`skills` 工具、安全模型）。本站权威，A2C-SMCP 反向引用。

    [:octicons-arrow-right-24: 进入 SKILL 协议](skill/index.md)

-   :material-server-network:{ .lg .middle } __MCP Server 配置__

    ---

    Plugin 内 `mcp-servers/` 目录约定 + A2C-SMCP v0.3.2 schema 字段速查（schema 权威在 A2C-SMCP）。

    [:octicons-arrow-right-24: 进入 MCP Server 配置](mcp-servers/index.md)

-   :material-store:{ .lg .middle } __Marketplace 分发__

    ---

    Plugin 包的 **Git 仓库分发约定**：`marketplace.json` / `plugin.json`、`source` 类型、Curator 模式、运行时行为参考。

    [:octicons-arrow-right-24: 进入 Marketplace 分发](marketplace/index.md)

</div>

## 与 A2C-SMCP 的边界

| 主题 | 权威 |
| --- | --- |
| SKILL 编写规范（内容契约） | **本站** SKILL 协议；[A2C-SMCP skill.md](https://doc.turingfocus.cn/a2c-smcp/latest/specification/skill/) 反向引用本站 |
| Plugin / Marketplace 路径配置规范 | **本站** Marketplace 规范 |
| MCP Server 配置 schema | [A2C-SMCP data-structures.md](https://doc.turingfocus.cn/a2c-smcp/latest/specification/data-structures/#mcp-server-配置结构) 权威；本站只做目录约定 + 速查 |
| SKILL 加载 / 使用 / 执行语义 | [A2C-SMCP 协议](https://doc.turingfocus.cn/a2c-smcp/latest/) 权威 |

## 设计参考

- [Anthropic Agent Skills 开放标准](https://agentskills.io/specification)
- [Claude Code Skills 文档](https://code.claude.com/docs/en/skills.md)
- [Claude Code Plugin Marketplaces](https://code.claude.com/docs/en/plugin-marketplaces.md)
- [Codex 文档站](https://learn.chatgpt.com/docs)

## 其他

<div class="grid cards" markdown>

-   :material-help-circle:{ .lg .middle } __常见问题__

    ---

    关于市场设计与使用的常见问题解答

    [:octicons-arrow-right-24: 查看 FAQ](appendix/faq.md)

</div>

## 版本信息

当前版本：**0.3.3**

## License

MIT License
