# 发布 Plugin

> 场景：把 SKILL / MCP Server 打包成 Plugin，通过 Git 仓库发布到 Marketplace，让 Robot 一键安装。
> 完整字段与 source 类型见 [Marketplace 规范](../marketplace/protocol.md)。

## 第 1 步：组织仓库目录

一个 Git 仓库 = 一个 Marketplace；仓库里可含多个 Plugin：

```
<repo>/                                  # = Marketplace
  .tfrobot-plugin/marketplace.json       # 必需：catalog 清单
  plugins/                               # 默认 plugin 根（可被 metadata.pluginRoot 覆写）
    <plugin-name>/
      .tfrobot-plugin/plugin.json        # plugin 清单
      skills/<skill-name>/SKILL.md       # SKILL 子树（可选）
      mcp-servers/<server>.json          # MCP 子树（可选）
      mcp-servers/inputs.json            # 输入定义（可选）
```

Plugin 至少含 `skills/` 或 `mcp-servers/` 之一非空；两者皆空视为空载。

## 第 2 步：写 plugin.json

```json
{
  "name": "data-toolkit",
  "version": "1.0.0",
  "description": "Data ingestion / cleansing / report skills",
  "author": { "name": "Acme Team", "email": "dev@acme.example.com" },
  "license": "MIT",
  "keywords": ["data", "etl"]
}
```

- `name`：Plugin 标识（kebab-case）；SKILL 暴露名以它为前缀（`<plugin>:<skill>`）。
- `version`：版本；更新时 bump 才能让已装用户收到新内容。

## 第 3 步：写 marketplace.json

```json
{
  "name": "acme-skills",
  "owner": { "name": "Acme Team", "email": "dev@acme.example.com" },
  "metadata": { "pluginRoot": "./plugins" },
  "plugins": [
    { "name": "data-toolkit", "source": "./plugins/data-toolkit", "description": "...", "version": "1.0.0", "strict": true }
  ]
}
```

- `source` 指向单个 plugin 目录（相对路径须 `./` 开头），或使用 `url` / `github` / `git-subdir` / `cnb` 引用外部 Git 仓库（curator 模式）。
- `strict: true`（默认）：plugin.json 是组件定义权威；`strict: false`：marketplace 条目是组件全集。
- 保留名空间：第三方 marketplace 不得占用 `tfrobot-` / `turingfocus-` / `tfs-` 前缀。

## 第 4 步：安装与更新

```text
/plugin marketplace add https://<git-host>/<group>/<repo>.git
/plugin install <plugin-name>@<marketplace-name>
```

- 更新：bump `plugin.json` 的 `version` 后，用户 `/plugin update` 即可拉取。
- 注意 v0.3.0 起 install / enable 分离：**install 只安装不激活**，需 `enable` 才点亮 SKILL 与 bundled server——「装了为什么没生效」的答案通常是尚未 enable（见 [加载行为参考 §10](../marketplace/loading-behavior.md#10-运行时状态层install--enable-分离v030-语义)）。

## 第 5 步：发布前校验

1. SKILL 内容契约自检（见 [编写一个 SKILL](write-a-skill.md) 第 5 步）。
2. 仓库校验脚本（镜像 A2C-SMCP SDK 校验边界）：`python3 -B scripts/validate_tfrobot_marketplace.py <仓库根>`。
3. 信息红线：无凭据、无真实 IP、无客户可识别信息。

## 参考

- [Marketplace 规范](../marketplace/protocol.md)（manifest schema 权威）
- [加载行为参考](../marketplace/loading-behavior.md)（运行时行为：strict mode、install/enable、版本优先级）
