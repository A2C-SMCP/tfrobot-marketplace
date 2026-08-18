# 使用指南

> 按目标找入口——先告诉我们要做什么，再进入对应指南。协议细节见右侧「协议规范」各章。

## 常见目标

<div class="grid cards" markdown>

-   :material-file-document-edit:{ .lg .middle } __我要编写一个 SKILL__

    ---

    给 Robot 写一个新能力包：命名、frontmatter、目录布局、占位符、安全边界。

    [:octicons-arrow-right-24: 编写一个 SKILL](write-a-skill.md)

-   :material-server-network:{ .lg .middle } __我要配置 MCP Server__

    ---

    在 Plugin 里给 SKILL 配上外部工具能力：传输模式、配置字段、输入定义。

    [:octicons-arrow-right-24: 配置 MCP Server](configure-mcp-server.md)

-   :material-store:{ .lg .middle } __我要发布 Plugin__

    ---

    把 SKILL / MCP Server 打包进 Plugin 并发布到 Marketplace：目录布局、manifest、版本与更新。

    [:octicons-arrow-right-24: 发布 Plugin](publish-plugin.md)

-   :material-sync:{ .lg .middle } __我要同时适配三个 Agent 系统__

    ---

    同一批能力适配 Claude Code / Codex / TFRobot：各平台差异、技术选型、等价移植。

    [:octicons-arrow-right-24: 三平台适配](three-platform-adaptation.md)

</div>

## 与 A2C-SMCP 的边界（先读这条）

TFRobot Marketplace 与 [A2C-SMCP 协议](https://doc.turingfocus.cn/a2c-smcp/latest/)是互补的两套文档：

| 主题 | 权威在哪 |
| --- | --- |
| **SKILL 编写规范**（内容契约：frontmatter / 目录 / 占位符） | 本站 [SKILL 协议](../skill/protocol.md)；A2C-SMCP [skill.md](https://doc.turingfocus.cn/a2c-smcp/latest/specification/skill/) 反向引用本站作为内容契约权威 |
| **Plugin / Marketplace 路径配置规范**（仓库目录布局、manifest） | 本站 [Marketplace 规范](../marketplace/protocol.md) |
| **MCP Server 配置 schema** | [A2C-SMCP data-structures.md](https://doc.turingfocus.cn/a2c-smcp/latest/specification/data-structures/#mcp-server-配置结构) 为权威；本站 [MCP Server 配置](../mcp-servers/protocol.md) 只规定目录约定 + 字段速查 |
| **SKILL 加载 / 使用语义**（staging、name 合成、`client:get_skill` 沙箱、错误码） | A2C-SMCP [skill.md](https://doc.turingfocus.cn/a2c-smcp/latest/specification/skill/) 权威 |
| **执行后端 / 沙箱 / 凭证基础设施** | A2C-SMCP 实施方文档；本站不规定 |

口诀：**写一个 SKILL 看本站；SKILL 怎么被 Computer 加载与执行看 A2C-SMCP。**
