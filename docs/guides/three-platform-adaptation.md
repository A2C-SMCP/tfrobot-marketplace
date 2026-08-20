# 三平台适配指南（Claude Code / Codex / TFRobot）

> 场景：同一批能力（SKILL 集合）要同时供 Claude Code、OpenAI Codex 与 TFRobot 使用。本指南给出三系统的差异对照与技术选型原则，供适配时逐项核对。
> 依据：各平台官方文档（2026-08 调研），链接见文末参考。

## 核心结论先行

1. **`SKILL.md` 是唯一跨三端复用的能力载体**——三系统都遵循 [Agent Skills 开放标准](https://agentskills.io/specification)（YAML frontmatter + Markdown body + `scripts/` / `references/` / `assets/`）。
2. **分发清单（plugin manifest / marketplace）与 SubAgent 都是平台私有**——没有跨系统等价物，各平台独立维护。
3. **能力契约落在 SKILL.md，SubAgent 只承载平台侧优化**——不要用平台私有的 SubAgent 定义承载跨端语义。

## 差异总表

| 维度 | Claude Code | Codex | TFRobot |
| --- | --- | --- | --- |
| SKILL 位置 | `.claude/skills/`（plugin 内 `skills/`） | `.agents/skills/`（plugin 内 `skills/`） | plugin 内 `skills/` |
| Plugin 清单 | `.claude-plugin/plugin.json` | `.codex-plugin/plugin.json` | `.tfrobot-plugin/plugin.json` |
| Marketplace | `.claude-plugin/marketplace.json` | `.agents/plugins/marketplace.json` | `.tfrobot-plugin/marketplace.json` |
| 安装 | `/plugin install <p>@<m>` | `codex plugin add <p>@<m>` | `/plugin install <p>@<m>` |
| 激活 | description 匹配自动加载 | `$` mention / `/skills` / description 隐式触发 | Robot 装配后进 LLM 工具循环 |
| **SubAgent** | `agents/*.md`（frontmatter：name/description/tools/model） | `.codex/agents/*.toml`（官方自述格式演进中） | **无此概念**——会话隔离由 Robot 平台承载 |
| 用户输入/凭据 | plugin.json `userConfig` + `${user_config.KEY}`（`sensitive` 入 keychain） | 无清单内凭据——从宿主环境继承 | `mcp-servers/inputs.json` + `${input:<id>}`（`password: true` 隐藏输入） |
| 密钥文件 | 无 | 无 | `.skillenv`（LLM 不可见，硬秘密边界） |
| 占位符 | `$ARGUMENTS` / `${CLAUDE_SKILL_DIR}` | 无 | `$TFROBOT_SKILL_DIR`（真实绝对目录）/ `$TFROBOT_SESSION_ID` / `$TFROBOT_ROBOT_ID` |
| frontmatter 扩展 | CLI 内支持扩展字段，但 claude.ai 上传 / Skills API 通道**硬报错** | 未知 key 忽略；额外读 `agents/openai.yaml` | **6 开放标准字段 + `tags`**（唯一平台私有字段，其余不解释） |
| 校验 | `claude plugin validate` | `codex plugin --help` 查原生 validate；否则官方 validator 脚本 | 仓库 `validate_tfrobot_marketplace.py`（镜像 SDK 边界） |

## 技术选型原则

### 1. SKILL.md：只用 6 个标准字段 + `tags`

`name`（1–64 字符，`[a-z0-9-]` kebab，**必须与目录名一致**）/ `description`（1–1024，首句触发关键词）/ `license` / `compatibility` / `tags`（TFRobot 平台私有：`list[str]`，0–10 条，单条 1–32 字符 kebab-case，[SKILL 协议 §3.3](../skill/protocol.md#33-tags-字段详述唯一平台私有字段)）/ `metadata` / `allowed-tools`。

- Claude Code 的扩展字段（`argument-hint` / `model` / `user-invocable` 等）CLI 内可用，但**出圈即报错**（claude.ai 上传 / Skills API / 打包分发校验器硬拒绝）。
- Codex 对未知 key 宽容（忽略），但为了三端一致同样只用 6 字段 + `tags`（Codex 侧忽略 `tags`，无副作用）。
- TFRobot 协议**私有字段仅 `tags`**（纯分类元数据；Claude Code / Codex 忽略未知 key，互操作不破坏）（[SKILL 协议 §0.3](../skill/protocol.md#03-与上游标准的关系)）。
- 若要兼容最窄通道（claude.ai 上传限 200 字符），description 控制在 200 字符内。

### 2. SubAgent：三系统最大差异，逐平台独立决策

| 平台 | 表达 | 注意 |
| --- | --- | --- |
| Claude Code | `agents/<name>.md`，`description` 写「何时应委派」（第三人称 + 触发词），`tools` / `disallowedTools` 白名单 | 插件内 agent 以 `<plugin>:<agent>` 暴露；每次 spawn 有 token 成本，需「verbose 输出 / 并行 / 工具限制 / 模型降级」之一正当化 |
| Codex | `.codex/agents/*.toml`（`name` / `description` / `developer_instructions`），或在 SKILL.md 内写 subagent workflow 指令 | 官方自述「格式可能演进、偏重」——**不作为跨平台长期资产**；plugin 不携带 `.codex/agents/` |
| TFRobot | 无 `agents/`——隔离需求写成「宿主支持的隔离 Robot 任务」表达 | 会话与线程隔离是 Robot 平台职责，SKILL 不声明执行编排 |

原则：**跨端契约只落在 SKILL.md；SubAgent 是平台侧可选优化**。若某平台暂时无法表达同等能力，在对应 SKILL 中写清降级行为，而不是保留无效的平台术语。

### 3. 凭据与用户输入：三种机制互不通用

- Claude Code：`plugin.json` 的 `userConfig` 声明字段，正文用 `${user_config.KEY}`；敏感字段 `sensitive: true` 存 keychain。
- Codex：**清单不写凭据**——spawn 时从宿主环境继承（团队密钥管理注入）。
- TFRobot：`mcp-servers/inputs.json` 声明输入（`promptString` / `pickString` / `command`），配置内用 `${input:<id>}`；密码标 `password: true`；SKILL 自身的密钥走 `.skillenv`（LLM 三层不可见）。

### 4. 平台专属能力：允许保留，写清理由

三平台不必逐字一致——业务目标、主要步骤、风险门控、验收标准一致即可；平台私有表达（工具名、交互方式、权限询问、MCP 配置、Agent/会话模型）必须本地化。平台专属 SKILL（如 Codex 的 slash 命令适配器）允许保留，在交付说明中记录。

## 适配工作流建议

1. **业务语义先行**：先定义与平台无关的统一语义（步骤、门控、验收），再逐平台落地。
2. **以 Claude Code 为语义参考**，移植时逐平台改写私有表达，不机械复制。
3. **同步范围先确认**：新增/修改 SKILL 后，明确要同步到哪些平台再动手；仅指源平台不算同步决定。
4. **逐平台校验**：三套校验各自跑通（见差异总表）。
5. **禁用捷径**：不要用软链接共享三套目录（Git clone / 缓存 / 打包行为不稳定）；不要为了文本一致把平台术语带进另一套产物。

## 参考

- [Claude Code docs map](https://code.claude.com/docs/en/claude_code_docs_map.md)（全站入口）· [skills](https://code.claude.com/docs/en/skills.md) · [sub-agents](https://code.claude.com/docs/en/sub-agents.md) · [plugin-marketplaces](https://code.claude.com/docs/en/plugin-marketplaces.md)
- [Codex 文档站](https://learn.chatgpt.com/docs)（llms.txt 全站索引：https://learn.chatgpt.com/llms.txt）· [build-skills](https://learn.chatgpt.com/docs/build-skills) · [build-plugins](https://learn.chatgpt.com/docs/build-plugins) · [subagents](https://learn.chatgpt.com/docs/agent-configuration/subagents) · [custom-prompts（已废弃）](https://learn.chatgpt.com/docs/custom-prompts)
- [Agent Skills 开放标准](https://agentskills.io/specification)（三端共同规范）
- TFRobot 侧：[SKILL 协议](../skill/protocol.md) · [Marketplace 规范](../marketplace/protocol.md) · [MCP Server 配置](../mcp-servers/protocol.md) · [A2C-SMCP 协议](https://doc.turingfocus.cn/a2c-smcp/latest/)
