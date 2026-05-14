# Marketplace 分发（TFRobotServer v1）

TFRobotServer Marketplace 定义 SKILL 包从 **Git 仓库** 到平台的分发流程，与 [Claude Code Plugin Marketplaces](https://code.claude.com/docs/en/plugin-marketplaces) 兼容并按 TFRobotServer 多租户、Curator 审核等需求做受控扩展。

对应 Story：[TFRS-201](https://turingfocus.atlassian.net/browse/TFRS-201)；Epic：[TFRS-179](https://turingfocus.atlassian.net/browse/TFRS-179)。

## 权威规范

| 文档 | 范围 |
| --- | --- |
| **[Marketplace v1 规范](protocol-v1.md)** ⭐ | Git 仓库分发标准 —— `marketplace.json` / `plugin.json` JSON Schema、`source` 类型枚举、目录布局、Curator 模式 |

新读者从此入手。**单个 SKILL 包的内容契约**（frontmatter / 占位符 / `skills` 工具等）见姊妹章节 [SKILL 协议](../skill/index.md)。

## 实施参考（运行时行为，非规范契约）

| 文档 | 范围 |
| --- | --- |
| [加载行为参考](loading-behavior.md) | Plugin manifest × strict mode 组合矩阵、缺失兜底、冲突检测 —— 基于 Claude Code 源码观察，给 Module D 实施工程师与 SKILL 作者参考 |
