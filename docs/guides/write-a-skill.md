# 编写一个 SKILL

> 场景：运营 / 工程师要为 Robot 写一个能力包（SKILL）。本指南走完「命名 → 内容 → 目录 → 占位符 → 安全边界」全流程。
> 详细字段约束见 [SKILL 协议](../skill/protocol.md)；SKILL 如何被 Computer 加载与执行见 [A2C-SMCP skill.md](https://doc.turingfocus.cn/a2c-smcp/latest/specification/skill/)。

## 第 1 步：确定命名与身份

- **目录名就是身份**：SKILL 暴露给 Agent 的名字取目录 basename（marketplace 源下为 `<plugin>:<skill>`）；改暴露名 = 改目录名。
- 命名规则：1–64 字符，`[a-z0-9-]`，kebab-case，不以 `-` 开头/结尾，无连续 `--`。
- 目录名必须等于 `SKILL.md` frontmatter 的 `name`。

```
my-skill/                  # ← 目录名 = 身份 = frontmatter name
├── SKILL.md               # 必需
├── references/            # 可选：LLM 渐进式读取的参考文档
├── scripts/               # 可选：Computer 侧可执行代码
├── assets/                # 可选：静态资源
└── .skillenv              # 可选：环境变量声明（LLM 全程不可见）
```

## 第 2 步：写 SKILL.md

frontmatter 共 6 个字段，全部来自 [Agent Skills 开放标准](https://agentskills.io/specification)（跨 Claude Code / Codex / TFRobot 互操作不破坏）：

```yaml
---
name: csv-aggregator
description: 把多个 CSV 文件按用户给定的列规则聚合并生成报告。当用户上传 CSV 并要求聚合/出报告时使用。
license: MIT
compatibility: 需要 Python 3.11+；预装 pandas
metadata: {}
allowed-tools: Bash
---
```

- `name`：必填，约束见第 1 步。
- `description`：必填，1–1024 字符。**「做什么 + 何时用」都写**，首句放核心触发关键词——它是 LLM 决定是否加载的唯一依据。
- 其余 4 个可选字段来自开放标准，TFRobot 平台不解释 `metadata`，跨客户端透传。

**不引入任何平台私有字段**（`model` / `runtime` / `hooks` 等一律不入）——执行后端差异由 A2C-SMCP 表达，不污染内容契约。完整不引入清单见 [SKILL 协议 §3.3](../skill/protocol.md#33-本规范不引入的字段完整列表)。

正文是 Markdown，由 LLM 直接消费：

- **分步骤写「如何执行」**，每步说明目标、动作、参考文件、预期输出。
- 引用包内文件用相对 SKILL.md 的路径（`references/foo.md`、`scripts/main.py`）。
- 正文控制在 ~500 行内，细节下沉 `references/`（渐进披露：body 只在触发时读入）。

## 第 3 步：声明环境与凭证（.skillenv，可选）

`.skillenv` 是标准 dotenv，位于包根，**LLM 全程不可见**：

```dotenv
LOG_LEVEL=INFO            # 字面量：直接作为环境变量传给执行环境
DATA_WAREHOUSE_TOKEN=     # 空值：从用户 vault 查询同名密钥；查不到则启动失败
```

- 它是**硬秘密边界**：任何读取请求都返回 403，内容永不进 LLM 上下文。
- 不支持 `"..."` 包裹、`${VAR}` 展开、多行值；KEY 重复报错。
- 注意：`.skillenv` 的解析与注入（vault 查询）目前标注「待实施」，边界语义先行（见 [SKILL 协议 §5](../skill/protocol.md#5-skillenv-环境变量声明)）。

## 第 4 步：使用占位符（可选）

正文可写 3 个 `TFROBOT_` 前缀占位符，在拼进 LLM prompt 前展开：

| 占位符 | 展开值 |
| --- | --- |
| `$TFROBOT_SKILL_DIR` | SKILL 包的**真实绝对目录**——Bash 可直接 `cd` / `open`（不是 URI） |
| `$TFROBOT_SESSION_ID` | 当前会话 UUID |
| `$TFROBOT_ROBOT_ID` | 当前 Robot 实例 ID |

未识别的 `$VAR` 保持字面透传；`$TFROBOT_` 前缀的未定义占位符报错。展开规则详见 [A2C-SMCP skill.md §9.4](https://doc.turingfocus.cn/a2c-smcp/latest/specification/skill/#94-占位符展开与目录路径可见性)。

## 第 5 步：安全自检（发布前必过）

1. `.skillenv` 与任何敏感值（token、vault 明文）不进 body、不进 prompt。
2. `scripts/` 只放可执行代码，不含凭据；执行后端（沙箱 / 引擎 / 凭证）由 Computer 侧 A2C-SMCP 决定，不在 SKILL 里声明。
3. LLM 读 SKILL 资源走内置 `skills` 工具（`skills(skill_name, path)`），无需写进 `allowed-tools`。

## 下一步

- 要给它配外部工具能力 → [配置 MCP Server](configure-mcp-server.md)
- 要发布 → [发布 Plugin](publish-plugin.md)
- 要同时适配 Claude Code / Codex → [三平台适配](three-platform-adaptation.md)
