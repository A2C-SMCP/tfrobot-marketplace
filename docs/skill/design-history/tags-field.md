# A6 — `tags` 分类元数据字段（SKILL v1 唯一平台私有 frontmatter 字段）

> 立项：[protocol#50](https://github.com/A2C-SMCP/a2c-smcp-protocol/issues/50)（协议侧需求与 wire 裁决）/ [marketplace#1](https://github.com/A2C-SMCP/tfrobot-marketplace/issues/1)（本仓 schema 定义）
> 范围：SKILL v1 frontmatter 新增 `tags` 字段；本规范「不引入独有字段」原则的首次修订。
> 结论一句话：**新增 `tags` 纯分类元数据字段（本平台唯一私有 frontmatter 字段）；协议侧 `A2CSkillRef.tags` 加性透传，Agent 全量获取后自行过滤。**

## 1. 背景与需求

A2C-SMCP SKILL 系统需要 Tag 标签能力用于分类 / 发现（protocol#50，v0.4.0）。经三方裁决：

- **不改任何协议事件**：`client:get_skills`、skill:// 资源机制、wire 事件全部不动；SKILL 量不大，使用方全量获取后使用时动态 Filter。
- **wire 落点**：`A2CSkillRef` 新增 `tags: NotRequired[list[str]]` 加性可选字段（与 `allowed_tools` 同款模式），进入 `client:get_skills` 响应。
- **schema 归属**：`tags` 的 schema 由 marketplace SKILL v1 单方主导，A2C 不重复定义——本仓负责字段形状 / 校验 / 词汇约定。

## 2. 裁定内容

| 项 | 裁定 | 理由 |
| --- | --- | --- |
| 类型 | `list[str]`（严格；不接受空格分隔字符串） | wire 形状即 `list[str]`；双形态无必要——分类元数据不承载工具许可语义（与 `allowed-tools` 定位不同） |
| 条目数 | 0–10 条（超限作者侧警告，多余条目仍透传） | 分类标签聚焦少数主题；上限防标签通胀 |
| 单条 | 1–32 字符；kebab-case 小写 `[a-z0-9-]` | 与 `name` 命名风格一致，便于统一过滤 / 展示 |
| 校验语义 | 非 `list[str]` → 省略字段 + 作者侧警告，SKILL 照常注册 | 与协议侧透传口径一致（protocol#50：省略 + 诊断日志，不触发任何加载拒绝路径） |
| 词汇约定 | v1 最小约定：kebab-case 小写、禁空串、不设保留字白名单 | 统一命名约定（大小写 / 保留字 / 空串语义）由 `ToolMeta.tags`（protocol#51）落地时给出规范表述，本规范承诺跟进对齐 |

## 3. 原则修订

A1 裁定「完全对齐开放标准 6 字段 + 不引入独有字段」。本次将 §1.1 原则 3 修订为「**不引入执行后端独有字段；唯一例外 `tags`（纯分类元数据）**」：

- **跨客户端互操作不破坏**：Claude Code / Codex 对未知 frontmatter key 忽略，`tags` 在非 TFRobot 客户端只是无声多余字段。
- **与 A3 `runtime` 撤回的差异**：`runtime` 承载执行后端语义（污染内容契约），`tags` 是纯元数据（不参与执行 / 授权决策），不违背原则 4「frontmatter 仅承载元信息」。

## 4. 与其他层 tags 的关系

| 层 | 载体 | 状态 |
| --- | --- | --- |
| plugin 级 | `marketplace.json` plugin 条目 `tags` | 已存在（[Marketplace 规范 §4.2](../../marketplace/protocol.md)） |
| **skill 级** | SKILL.md frontmatter `tags` | **本裁定新增** |
| MCP Tool 级 | `ToolMeta.tags`（Server 声明 + Computer 配置三层合并） | [protocol#51](https://github.com/A2C-SMCP/a2c-smcp-protocol/issues/51) 在推；词汇约定对齐锚点 |

三层粒度不同、互不替代；skill 级 tags 随 `A2CSkillRef` 进入 Agent 侧过滤。

## 5. 对齐承诺

`ToolMeta.tags`（protocol#51）的统一词汇约定（大小写 / 保留字 / 空串语义）落地后，**以协议仓为权威**，本规范跟进对齐，避免两套 tag 体系各自演化。
