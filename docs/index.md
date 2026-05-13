# TFRobot Marketplace

**TFRobot Marketplace** 是 TFRobot 生态的市场平台，承载第三方扩展、Skill、Computer 配置等内容的发布、发现与分发流程。本文档站点沉淀市场的开放接口、数据规范、接入指南等内容。

## 两大支柱

<div class="grid cards" markdown>

-   :material-puzzle:{ .lg .middle } __SKILL 协议__

    ---

    单个 SKILL 包的 **内容契约**：frontmatter 字段、目录布局、`runtime` 枚举、占位符、`skills` 工具、安全模型。

    [:octicons-arrow-right-24: 进入 SKILL 协议](skill/index.md)

-   :material-store:{ .lg .middle } __Marketplace 分发__

    ---

    SKILL 包的 **Git 仓库分发约定**：`marketplace.json` / `plugin.json` JSON Schema、`source` 类型、Curator 模式，兼容 Claude Code Plugin Marketplaces。

    [:octicons-arrow-right-24: 进入 Marketplace 分发](marketplace/index.md)

</div>

## 设计参考

- [Anthropic Agent Skills 开放标准](https://agentskills.io/specification)
- [Claude Code Skills 文档](https://code.claude.com/docs/en/skills)
- [Claude Code Plugin Marketplaces](https://code.claude.com/docs/en/plugin-marketplaces)

对应 Epic：[TFRS-179](https://turingfocus.atlassian.net/browse/TFRS-179)。

## 其他

<div class="grid cards" markdown>

-   :material-help-circle:{ .lg .middle } __常见问题__

    ---

    关于市场设计与使用的常见问题解答

    [:octicons-arrow-right-24: 查看 FAQ](appendix/faq.md)

</div>

## 版本信息

当前版本：**0.1.0**

## License

MIT License
