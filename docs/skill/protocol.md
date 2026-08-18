# TFRobot SKILL 协议规范

> 状态：**规范定稿** —— A1\~A4 分步设计文档（见 [设计史](design-history/index.md)）整合而成的单一权威契约。
> 范围：**SKILL 撰写规范与使用语义**——内容契约（如何写一个 SKILL）+ 加载使用约束（任何接入方必守）。**不涉及** SKILL 分发渠道、运行时存储介质、执行后端（沙箱 / 引擎 / 容器 / 凭证基础设施）等业务侧议题——这些由接入方业务库 / Computer 侧 A2C-SMCP 实施方文档处理。
> 读者：**SKILL 作者**（开发指南）+ **接入方实施工程师**（加载使用层的最小协议契约）。
> 动手前先看：[编写一个 SKILL](../guides/write-a-skill.md)（场景化流程）；本章是字段级契约。

## 0. 总览

### 0.1 这是什么

TFRobot SKILL 协议定义**单个能力包（SKILL）的撰写规范与使用语义**。

**SKILL 是什么**：

* **一个文件夹**，包含一份 `SKILL.md`（frontmatter + body）+ 可选的脚本、参考资料、资源文件
* **平台无关** —— 协议本身**不指定**分发渠道、存储介质、执行后端；同一个 SKILL 文件夹可在 Claude Code / Cursor / TFRobot 等任何兼容客户端加载使用
* TFRobot 数字员工通过加载 SKILL 来扩展能力 —— 与 Anthropic [Agent Skills 开放标准](https://agentskills.io/specification)中的 Skill 概念**等价**

**协议层扩展概览**：相对上游标准，本协议在 `.skillenv` 双语义（§5）、`skills` 工具（§7）、`$TFROBOT_*` 占位符（§6）、SKILL 资源访问安全原则（§8）等处做了受控扩展，以适配 Robot ↔ Computer 的混合执行模型。**本规范不引入任何独有 frontmatter 字段**——6 个 frontmatter 字段全部来自 Agent Skills 开放标准。

### 0.2 SKILL 归属与来源

**SKILL 归属于 Robot**——切换 Robot 即切换可见 SKILL 集合。这是用户与接入方都应建立的基本心智模型：SKILL 不是孤立资产，而是 Robot 当前能力面的一部分。

**Robot 的 SKILL 集合由多来源汇聚而成**，且**来源决定执行能力**：

| 来源 | 内容 | 执行能力 | 协议层关心 |
| --- | --- | --- | --- |
| **云端 Robot 配置** | 从云端对象存储的约定目录提取——按 SKILL `name` 在约定根目录下查找子目录（默认根 `skills/`；TFRobot 平台采用 MinIO 实现）。具体存储后端、路径前缀、命名空间隔离策略由接入方业务库文档定义 | **仅文档提示**——LLM 在工具循环中读 `references/` 并按 body 指示行事；**不执行任何脚本** | 内容契约 + LLM 与 SKILL 的交互（§4 / §6 / §7） |
| **已连接的 Computer** | Computer 按 [A2C-SMCP 协议](https://github.com/A2C-SMCP)（独立规范）暴露本地可用 SKILL；Robot 通过 A2C 连接发现并挂载 | **可执行**——`scripts/` 内容在 Computer 侧运行；执行后端（沙箱 / 引擎 / 容器 / 凭证）完全由 Computer 决定，本协议不规定 | 内容契约（同上）+ `.skillenv` 等环境声明（§5），由 Computer 侧解释 |
| **Marketplace 仓库** | Git 仓库分发，详见独立规范 [Marketplace 规范](../marketplace/protocol.md) | 取决于挂载点：作为 Robot 配置导入 = 仅文档；作为 Computer 侧能力提供 = 可执行 | 同上 |

**协议边界**：本协议**只规定** SKILL 文件夹本身的内容契约（如何撰写、如何加载、如何使用），以及 LLM 与 SKILL 资源交互的最小契约（§7 `skills` 工具）。
- SKILL 从何处来、在何处落盘、跨进程如何同步——由接入方业务库文档定义
- SKILL 的脚本如何执行、用什么沙箱 / 容器 / 引擎、凭证从何而来——由 Computer 侧 A2C-SMCP 实施方文档定义

> 即使来源不同，落到加载方手中的都是**等价的 SKILL 文件夹**；其后续装载、渲染、LLM 交互走完全相同的协议层路径。可执行部分则由 Computer 侧按 A2C-SMCP 自定义实现。

### 0.3 与上游标准的关系

* **完全对齐**[Agent Skills 开放标准](https://agentskills.io/specification)的 6 个 frontmatter 字段（`name` / `description` / `license` / `compatibility` / `metadata` / `allowed-tools`），跨 Claude Code / Codex / Cursor / Goose 等兼容客户端**互操作不破坏**。
* **不采纳**任何 Claude Code 私有扩展字段（CLI 残留 / 平台决策项 / 本地 FS 概念）。
* **不引入任何独有 frontmatter 字段**——执行后端差异由 Computer 侧 A2C-SMCP 协议表达，不污染 SKILL 内容契约。

### 0.5 与 A2C-SMCP 协议的边界（先读这条）

| 主题 | 权威 | 具体地址 |
| --- | --- | --- |
| SKILL 编写规范（本章全部内容） | **本站** | A2C-SMCP [skill.md](https://doc.turingfocus.cn/a2c-smcp/latest/specification/skill/) 明确「SKILL = 符合 marketplace SKILL v1 规范的目录包」，**反向引用本站**为内容契约权威 |
| SKILL 的 staging 物化 / name 合成 / 三形态（`<plugin>:<skill>` 等） | A2C-SMCP | [skill.md §1 命名](https://doc.turingfocus.cn/a2c-smcp/latest/specification/skill/#1-skill-命名) |
| `client:get_skill` 读取沙箱 / `.skillenv` 4017 边界 / 错误码 | A2C-SMCP | [skill.md §9-§10](https://doc.turingfocus.cn/a2c-smcp/latest/specification/skill/)、[error-handling.md](https://doc.turingfocus.cn/a2c-smcp/latest/specification/error-handling/) |
| `${TFROBOT_SKILL_DIR}` 展开规范（render-time / 真实绝对目录） | A2C-SMCP | [skill.md §9.4](https://doc.turingfocus.cn/a2c-smcp/latest/specification/skill/#94-占位符展开与目录路径可见性)（本章 §6.1 为其摘要） |
| MCP Server 配置 schema | A2C-SMCP | [data-structures.md § MCP Server 配置结构](https://doc.turingfocus.cn/a2c-smcp/latest/specification/data-structures/#mcp-server-配置结构) |
| 执行后端 / 沙箱 / 凭证基础设施 | A2C-SMCP 实施方文档 | 本章不规定 |

口诀：**写一个 SKILL 看本站；SKILL 怎么被 Computer 加载与执行看 A2C-SMCP。**

### 0.4 范围与不范围

**本规范收入**：

* Frontmatter 6 字段（全部来自开放标准）+ 标准目录结构 + `.skillenv` 双语义环境变量声明 + 3 个运行时占位符 + 平台内置 `skills` 工具 + 3 条 SKILL 资源访问安全原则。

**本规范不收入**（详见 §10）：

* CLI 残留占位符 / 平台决策性 frontmatter 字段（model / effort / disable-model-invocation 等）/ 完整性签名校验 / Web 编辑器 / 调试器 UI / 资源配额计费 / 并发隔离精细策略。
* **`runtime` 字段、Dockerfile 文件名约定、引擎枚举、沙箱 I/O 协议**——执行后端归 Computer 侧 A2C-SMCP，本协议不规定。
* **不规定**：SKILL 分发渠道实施、运行时存储介质、密钥基础设施实现、跨进程同步机制等业务侧议题。

## 1. 设计原则

本规范设计共 9 条原则：6 条来自 A1~A4 的"克制 + 互操作 + 安全"基调；3 条是 SKILL 资源访问安全模型的强制约束。

### 1.1 协议层（6 条）

1. **完全对齐 Agent Skills 开放标准**：6 个 frontmatter 字段全采纳，不破坏跨客户端互操作。
2. **不采纳 Claude Code 扩展字段**：CLI 残留（`$ARGUMENTS` / `arguments` / `argument-hint`）、平台决策项（`model` / `effort` / `disable-model-invocation` / `user-invocable`）、本地 FS 概念（`hooks` / `paths`）等一律不入；理由汇总在 A1 §3。
3. **不引入独有字段**：执行后端差异（沙箱 / 引擎 / 容器选择）由 Computer 侧 A2C-SMCP 表达；SKILL frontmatter **不承载**该类信息。设计史中 A3 曾设计 `runtime` 字段，最终撤回（见附录 A）。
4. **可由仓库内文件表达的不入 frontmatter**：环境变量 → `.skillenv`；frontmatter 仅承载元信息。
5. **占位符私有命名空间**：所有运行时占位符以 `TFROBOT_` 前缀；不 mirror Claude Code `CLAUDE_*` / `$ARGUMENTS`；未识别占位符跨平台加载时字面透传。
6. **克制**：所有候选字段 / 占位符 / 取值均需具体 SKILL 场景驱动；不预设未实证的能力。

### 1.2 SKILL 资源访问安全原则（3 条强制）

SKILL 内容来自不受加载方控制的多个来源（云端 Robot 配置 / Computer 上报 / Marketplace 仓库），**作为不可信资产对待**。任何接入方实施必须满足以下三条：

7. **SKILL 资源访问受控**：所有 SKILL 文件访问（LLM 读 `references/` / `scripts/`）都必须经由受控通道——LLM 端走平台内置 `skills` 工具（§7）。**不允许**将 SKILL 原始存储凭据下放给 LLM，也不允许接入方把 SKILL 当本地代码 import / eval。Computer 侧执行 `scripts/` 时的资源访问由 A2C-SMCP 自行约束。
8. **加载方不持久化 SKILL**：SKILL 字节**只**在加载方进程内存中一次性使用（喂给 LLM 上下文 / 通过 A2C 传递给 Computer），**不**落盘到加载方进程 FS、**不**派生子进程执行 SKILL 衍生命令、**不**基于 SKILL 文件 FS 权限做任何决策。SKILL 对加载方而言**只是 data**。
9. **敏感凭证 LLM 不可见**：`.skillenv` 解析后的明文及任何由其引用的私人 vault 内容**不进入** SKILL.md body 展开路径、**不进入** LLM 上下文（详见 §5.4 / §8）。

## 2. SKILL 包结构

```
my-skill/                          # 包根目录名 = SKILL.md frontmatter `name`
├── SKILL.md                       # 必需 —— frontmatter + body
├── .skillenv                      # 可选 —— 环境变量声明（§5）
├── scripts/                       # 标准命名 —— Computer 侧可执行代码（Robot 配置侧 SKILL 可忽略）
├── references/                    # 标准命名 —— LLM 渐进式读取的参考文档
├── assets/                        # 标准命名 —— 静态资源
└── <任意其他目录或文件>             # 作者自由扩展，平台不解释
```

### 2.1 强约束

| 项 | 约束 |
| --- | --- |
| 包根目录名 | 必须等于 `SKILL.md` frontmatter 中的 `name`。**目录 basename 是身份**（见下方 note） |
| `SKILL.md` | 必须存在于包根；缺失即加载失败 |
| 受协议解释的 hidden-file | 仅 `.skillenv`；其他 `.xxx` 平台不解释也不拒绝 |
| `scripts/` / `references/` / `assets/` | 命名标准化但**不强制必填** —— 按需创建即可，缺失加载器不报错 |

!!! note "目录 basename 是身份（按来源分三态）"

    暴露给 Agent 的 SKILL name 由**目录 basename** 决定，而非 frontmatter `name`——frontmatter `name` 在多数来源下仅作显示名。作者要改暴露名，须**改目录名**（并保持 frontmatter `name` 一致）。

    | 来源 | 身份规则（SDK staging 实测） |
    | --- | --- |
    | marketplace | name = `<plugin>:<目录 basename>`；frontmatter.name 仅作显示名、不改 ID（防伪） |
    | user（手动 drop-in） | name = 目录 basename（单段裸名）；frontmatter.name 不一致仅记 DEBUG |
    | mcp（Computer 从 MCP server 物化） | 才真正把目录重命名为 frontmatter.name |

    跨工具可见名规则（A2C skill.md §1 权威）：marketplace 源 `<plugin>:<skill>`、user 源裸 `<skill>`、mcp 源 `mcp:<bundle_id>:<skill>`——**不含 marketplace 名**。

### 2.2 语义化命名指引（建议遵循）

| 目录 | 语义 |
| --- | --- |
| `scripts/` | Computer 侧可执行代码（`.py` / `.ts` / `.js` / shell 等）。Robot 配置侧 SKILL 中此目录无运行语义，仅作为 LLM 可读资源参考 |
| `references/` | LLM 渐进式读取的参考文档（Markdown / 纯文本） |
| `assets/` | 静态资源（图片、字体、文档模板、二进制等） |

## 3. SKILL.md Frontmatter 字段表

YAML frontmatter 位于 SKILL.md 顶部，包裹在 `---` 之间。共 6 个字段，全部来自 Agent Skills 开放标准。

### 3.1 必填字段

| 字段 | 类型 | 约束 | 含义 |
| --- | --- | --- | --- |
| `name` | string | 1–64 字符；`[a-z0-9-]`；不以 `-` 开头/结尾；无连续 `--`；**与目录名一致** | SKILL 唯一标识 |
| `description` | string | 1–1024 字符；非空 | "做什么 + 何时用"；Robot 启动时进入 LLM 可见 skill 列表（progressive disclosure 第一层）；**首句包含核心触发关键词** |

### 3.2 可选字段（开放标准）

| 字段 | 类型 | 约束 | 含义 |
| --- | --- | --- | --- |
| `license` | string | 标准未规定字符上限 | 许可声明 |
| `compatibility` | string | 最长 500 字符 | 人类可读环境要求 hint（如"需要互联网访问"、"需要 Computer 预装 pandas"） |
| `metadata` | map<string,string> | 标准未规定大小上限 | 自由 key-value，跨客户端互操作透传；TFRobot 平台**不解释** |
| `allowed-tools` | string \| list | 空格分隔字符串或 YAML 列表 | LLM 推理循环中可直接调用、免授权的工具白名单；**与 Computer 侧执行无关**，仅作用于 LLM-工具循环层面 |

### 3.3 本规范不引入的字段（完整列表）

CLI 残留：`arguments` / `argument-hint` / `$ARGUMENTS` / `$N` / `$name`
平台决策：`model` / `effort` / `disable-model-invocation` / `user-invocable` / `agent`
本地 FS 概念：`hooks` / `paths`
环境隐式：`shell`
执行后端：`runtime` / `dockerfile` / `image` / `byoi`（执行后端由 Computer 侧 A2C-SMCP 表达）
候选未引入：`version` / `secrets` / `network` / `egress` / `visibility` / `audience` / `quota` / `cost` / `signature` / `integrity`

理由汇总参见 A1 §3 与附录 A（A3 撤回 rationale）。

## 4. SKILL.md Body 约定

* **格式**：Markdown，由 LLM 直接消费（拼到 prompt 中）
* **相对路径引用**：参照 Agent Skills 开放标准 —— 引用包内文件用"相对 SKILL.md 的路径"（如 `scripts/main.py` / `references/foo.md`）；具体由平台 Read 工具（§7）按"当前激活 SKILL"上下文解析
* **占位符**：SKILL.md body 中的 `$TFROBOT_*` 占位符在激活时由平台展开（§6.2）
* **触发表述**：description 的首句必须包含核心触发关键词；body 可详述步骤、调用工具、引用资源
* **执行指引**：body 可指示 LLM "调用 Computer 侧暴露的脚本 X"；具体调用语义走 A2C-SMCP 工具调用协议，本规范不展开

## 5. `.skillenv` 环境变量声明

仓库根可选文件，声明 SKILL 执行所需的环境变量与密钥来源。**LLM 全程不可见**。

> 协议层只规定 `.skillenv` 的**语法与语义**。Robot 配置侧 SKILL 不执行脚本，因此 `.skillenv` 主要服务 Computer 侧 SKILL。
>
> **实施状态（待实施）**：当前双 SDK 对 `.skillenv` **仅作硬秘密边界**——任何 `rel_path` 命中即 `4017 forbidden`、不泄漏存在性，**没有任何 vault 查询 / 执行环境注入实现**；A2C skill.md 亦只把 `.skillenv` 定义为硬秘密边界（§9.1 原则三）。本节「解析 + 注入」的语义归属哪一侧（Agent SDK / Robot）目前是无实现支撑的约定，接入方实施前需先落定归属。

### 5.1 文件契约

| 项 | 值 |
| --- | --- |
| 文件名 | `.skillenv` |
| 位置 | skill 包根目录，与 `SKILL.md` 同级 |
| 是否必填 | 可选 |
| 文件格式 | 标准 [dotenv](https://github.com/motdotla/dotenv)（每行 `KEY=VALUE`，`#` 行注释，空行允许） |
| 编码 | UTF-8（无 BOM），LF 换行 |

### 5.2 双语义（dotenv 标准语义之上的唯一约定）

| 行格式 | 语义 |
| --- | --- |
| `KEY=VALUE`（VALUE 非空） | **字面量** —— VALUE 整段作为 env var 值传给执行环境；不做任何解析或变量替换 |
| `KEY=`（VALUE 为空） | **用户 vault 引用** —— 以 KEY 为名查询当前用户的私人凭证 vault；查到 → 注入解析后的明文；查不到 → 执行启动失败 |

> "用户 vault" 是一个抽象概念，指代由接入方（云端服务 / Computer 侧）维护、按当前用户上下文解析的密钥源。本协议不规定具体后端实现。

### 5.3 行格式规则

| 规则 | 内容 |
| --- | --- |
| R1 | KEY 形如 `[A-Z_][A-Z0-9_]*`（POSIX env var 名约定） |
| R2 | 非空 VALUE → 字面量传递 |
| R3 | 空 VALUE → 用户 vault 同名查询 |
| R4 | `#` 起始行为注释；**行尾注释不支持** |
| R5 | 空行允许 |
| R6 | **不支持** `"..."` / `'...'` 包裹，**不支持** `${VAR}` / `$VAR` 展开 |
| R7 | KEY 重复 → 解析报错 |
| R8 | **不支持**多行 VALUE |

### 5.4 LLM 不可见性（三层防护，接入方必守）

1. **加载层黑名单**：SKILL 加载器维护 LLM 不可见文件名集合（至少含 `.skillenv`）；LLM 任何路径请求都必须拒绝并记审计日志
2. **执行注入而非文件分发**：`.skillenv` 解析必须在执行环境创建前完成，通过执行环境的 env 注入通道传入；**原文件不进入执行环境 FS**
3. **prompt 渲染产物守护**：接入方应在 CI 中断言最终拼到 LLM 的 prompt 字符串不包含 `.skillenv` 任何行内容（KEY 名、VALUE 字面量、注释）

### 5.5 与平台密钥基础设施的边界（协议层约束）

`.skillenv` 仅服务**用户私人凭证 vault**。协议层强制：

| 维度 | 约束 |
| --- | --- |
| 协议层语法 | `.skillenv` 没有任何语法能引用接入方的平台共享密钥（如平台付费配额、共享 API key 池等） |
| 解析层映射 | 空 VALUE 时**只**查询用户私人 vault，不得回退到平台共享密钥源 |
| 注入层目标 | 执行环境 env 只含用户 vault 解析结果 + `.skillenv` 字面量；**平台共享密钥永不出现在执行环境 env** |

> 背景：若把平台共享密钥注入 `.skillenv`，用户脚本一句 `print(os.environ)` 就能 dump 出整个平台付费配额。协议层、解析层、注入层**三处强制隔离**。具体凭证基础设施实现（vault 后端、密钥缓存、传输通道等）由接入方业务文档定义。

## 6. 占位符

### 6.1 占位符全集（3 个）

全部以 `TFROBOT_` 前缀；SKILL.md body 中可写 `$TFROBOT_<NAME>` 或 `${TFROBOT_<NAME>}`（等价）。

| 占位符 | 语义 |
| --- | --- |
| `$TFROBOT_SKILL_DIR` | 当前 SKILL 包的**真实绝对目录**（= A2C `A2CSkillRef.path`）。渲染期展开为本地路径，Bash 可直接 `cd` / `open`；**不得**展开为 `skill://` 不透明 URI（A2C skill.md §9.4 规范权威，见下方 note） |
| `$TFROBOT_SESSION_ID` | 当前会话 UUID |
| `$TFROBOT_ROBOT_ID` | 当前 Robot 实例 ID |

!!! note "`$TFROBOT_SKILL_DIR` 展开目标：真实绝对目录（非 URI）"

    A2C-SMCP skill.md §9.4 是本节占位符展开的**规范权威**：

    1. **占位符集是闭合白名单**：未在白名单内的 `${...}` 与裸 `$` 文本 MUST 原样透传（SKILL.md 正文里的 shell 变量 / `$` 金额不被误伤）。
    2. **展开时机 = render-time**：Agent SDK 在把内容拼进 LLM prompt **之前**替换；`client:get_skill` 只投递原始字节、占位符不展开（`total_size` / `sha256` 基于未展开字节）。
    3. **展开目标 MUST 为真实绝对目录**（= `A2CSkillRef.path`），**不得**展开为 `skill://` 不透明 URI——Bash 无法 `cd` 进 URI；且 URI 仅 MCP 源存在，对 marketplace / user 源根本不可用。稳定标识需求由协议主键 `name` 满足。早期「对 LLM 展开为不透明 URI」的表述已被 A2C **主动撤回**（消除「远程不可信」姿态残留）。
    4. **子进程 env 注入是运行期防御纵深**：Computer 执行 `scripts/` 子进程时 MAY 额外注入同名 env `TFROBOT_SKILL_DIR`（值 = skill 绝对目录），供脚本运行期 `os.environ[...]` 自引用。它是子进程 env，**不是** body 文本渲染机制——两者作用对象不同。
    5. **秘密边界不变**：泄露 skill 目录路径 ≠ 泄露秘密；`.skillenv` 仍是硬秘密边界（§5），任何 `rel_path` 都不可读出、注入时不写日志、不进 prompt。

> `$TFROBOT_SKILL_DIR` 字面暴露给 LLM **不构成漏洞** —— 目录路径本身藏不住（LLM 本就能跑 `pwd` / `ls`），真正的秘密由 `.skillenv` 边界独立守护（§5 / §8）。

> Computer 侧若选择将这些占位符也注入脚本执行环境（如 `os.environ["TFROBOT_SKILL_DIR"]`），命名仍遵循"去掉 `$` / `${}`"约定；具体注入由 A2C-SMCP 实施方负责（见上方 note 第 4 条）。

### 6.2 展开规则

| 项 | 规则 |
| --- | --- |
| 时机 | 平台在拼到 LLM prompt **之前**完成替换；LLM 见到的是已展开值 |
| 语法 | `$TFROBOT_NAME` 与 `${TFROBOT_NAME}` 等价 |
| `${TFROBOT_SKILL_DIR}` 展开目标 | **真实绝对目录**（非 URI）——Bash 可 `cd`；稳定标识需求由协议主键 `name` 满足（A2C skill.md §9.4(3)） |
| 命名空间 | 仅识别 `$TFROBOT_*` 开头；其他 `$VAR` 保持字面原样 |
| 未定义占位符 | `$TFROBOT_FOOBAR` 等 → 加载器报错 |
| 转义 | 不支持；需要字面字符串用 code block 或文本说明 |

### 6.3 不收入的占位符（与不引入理由）

| 候选 | 不收入理由 |
| --- | --- |
| `$ARGUMENTS` / `$N` / `$name` | CLI 残留（A1 §3.1 已排除） |
| `$TFROBOT_API_ENDPOINT` / `$TFROBOT_AUTH_TOKEN` | 不引入回调通道；需要时由 Computer 侧 A2C-SMCP 自行表达 |
| `.skillenv` 解析后的用户 vault 明文 | 敏感值隔离 —— 仅作为执行环境 env 注入，不在 body 展开 |
| `$TFROBOT_TENANT_ID` / `$TFROBOT_USER_ID` | 多租户上下文不暴露；按 tenant 区分应在 robot 配置层 |
| `$TFROBOT_ARG` / `$TFROBOT_INPUT` 类输入占位符 | 输入参数注入语义涉及 LLM 工具调用协议；由 A2C-SMCP 表达 |
| `$TFROBOT_EFFORT` / `$TFROBOT_MODEL` | 模型与算力档由平台决定，不暴露 |
| `$TFROBOT_REQUEST_ID` | 可观测性走平台日志 / trace |

## 7. 平台内置 `skills` 工具

**LLM 读 SKILL 资源的唯一受控通道**（§1.2 第 7 条）。任何 SKILL 激活期间隐式可用，不需作者写入 `allowed-tools`，也不能被禁用。

### 7.1 工具契约

| 项 | 设计 |
| --- | --- |
| 工具名 | `skills`（与 Claude Code 命名一致） |
| 入参 1 | `skill_name`: string —— 平台已注册 SKILL 的 `name`（§3.1） |
| 入参 2 | `path`: string —— 相对 skill 根的路径（如 `references/foo.md` / `scripts/main.py` / `assets/template.docx`） |
| 返回 | 文件内容（文本直返；二进制走 base64） |
| **黑名单（强制 403）** | `.skillenv` / 任何 `.skillenv*` 模式 |
| **路径校验** | 拒绝 `..` / 绝对路径 / 跳出 skill 根 / 符号链接逃逸 |
| **授权校验** | 当前会话有权访问 `skill_name` 才允许；权限模型由接入方 registry 定 |
| **大小上限** | Robot 工具实现侧建议 ≤ 10 MB（超过返回错误并提示走 Computer 侧执行处理）。若本工具转发到 Computer 侧 `client:get_skill`，作者实际感受到的是 **Computer 阈值**：文本 ≤ 32 KiB 内联 body、超则转 blob 句柄；100 MiB 硬上限（`A2C_SKILL_INLINE_BUDGET` / `A2C_SKILL_MAX_SIZE` 可调） |
| **作用域** | 仅 LLM 工具调用层；Computer 侧脚本访问自身 SKILL 资源走 A2C-SMCP，不通过本工具 |
| **跨 SKILL 读** | 天然支持 —— `skill_name` 是显式入参；授权由 registry 把关 |

> **黑名单与 Computer 侧的差异**：本工具契约的 `.skillenv*` 模式比 Computer 侧更严——A2C `client:get_skill` 只精确匹配 basename `.skillenv`（含 symlink realpath 复检，命中 → `4017 forbidden`）。更严无冲突（本工具是 TFRobot 平台自己的工具契约），但实施转发时注意两套黑名单各自生效。

### 7.2 典型调用

```python
# LLM 在 Robot 配置侧 SKILL 中按 body 指示读取参考文档
skills(skill_name="deploy-helper", path="references/checklist.md")

# 跨 SKILL 引用共享资源（如 shared template / 术语表）
skills(skill_name="common-utils", path="references/glossary.md")

# 黑名单触发拒绝
skills(skill_name="csv-aggregator", path=".skillenv")
# → 403 access denied: protected file
```

## 8. 安全模型（汇总）

本节是 §1.2 / §5.4 / §5.5 / §7.1 中安全相关约束的索引；接入方在 review 实施时使用。

### 8.1 三条强制原则

| 原则 | 协议层落位 |
| --- | --- |
| **SKILL 资源访问受控** | LLM 走 `skills` 工具（§7）；Computer 侧脚本走 A2C-SMCP 自身受控通道 |
| **加载方不持久化 SKILL** | 只读字节、不下盘、不 `subprocess` 执行 SKILL 衍生命令、不基于 SKILL 文件 FS 权限做决策；当 data 喂 LLM 一次性使用 |
| **敏感凭证 LLM 不可见** | `.skillenv` 三层防护（§5.4）；body 中不展开 vault 明文 |

### 8.2 LLM 不可见性保障

`.skillenv` 与所有敏感值（短期 auth token、用户 vault 明文）必须满足：

* 加载器黑名单拒绝任何 path 请求（§5.4 第 1 条）
* 执行注入而非文件分发（§5.4 第 2 条）
* CI 守护 prompt 渲染产物（§5.4 第 3 条）

### 8.3 与平台密钥基础设施的信任隔离

`.skillenv` 仅服务**用户私人凭证 vault**；接入方的平台共享密钥（付费 API quota、共享 service account 等）必须在协议层、解析层、注入层三处强制隔离，物理上无任何语法能引用（§5.5）。

## 9. 完整示例

### 9.1 Robot 配置侧 SKILL（仅文档驱动，无脚本）

包目录：

```
deploy-helper/
├── SKILL.md
└── references/
    └── checklist.md
```

`SKILL.md`：

```yaml
---
name: deploy-helper
description: 帮助用户准备发布到目标环境的清单。当用户说"准备发布"或类似意图时触发。
---

## 任务流程

1. 询问用户目标环境（dev / staging / prod）与版本号
2. 调用 `skills(skill_name="deploy-helper", path="references/checklist.md")` 读取按环境分组的检查项
3. 根据当前 robot 状态逐项核对
4. 输出报告，会话 ID 引用 `$TFROBOT_SESSION_ID` 以便后续追溯
```

> Robot 配置侧 SKILL **不执行任何脚本**；LLM 在工具循环中读 `references/` 并按 body 指示行事。`$TFROBOT_SESSION_ID` 在激活时展开为具体 UUID。

### 9.2 Computer 侧 SKILL（含脚本，由 A2C-SMCP 暴露）

包目录：

```
csv-aggregator/
├── SKILL.md
├── .skillenv
├── scripts/
│   └── main.py
├── references/
│   └── column-mapping.md
└── assets/
    └── report-template.docx
```

`SKILL.md`：

```yaml
---
name: csv-aggregator
description: 把多个 CSV 文件按用户给定的列规则聚合，并按 assets/report-template.docx 模板生成报告。当用户上传 CSV 并要求聚合/出报告时触发。
compatibility: 需要 Computer 侧 Python 3.11+ 环境；预装 pandas / python-docx；列规则参见 references/column-mapping.md
---

## 执行

1. LLM 通过 A2C-SMCP 调用 Computer 侧暴露的 `csv-aggregator` 工具
2. Computer 侧加载本 SKILL 包，按其内部约定执行 `scripts/main.py`（执行后端 / 沙箱 / 资源限额由 Computer 决定）
3. 脚本把处理统计 JSON 写到 stdout；LLM 收到 stdout 作为 tool result
```

`.skillenv`：

```dotenv
LOG_LEVEL=INFO
# 空 VALUE 即触发用户 vault 查询；下行示例（按需启用）
# DATA_WAREHOUSE_TOKEN=
```

`scripts/main.py`（由 Computer 侧执行，环境变量由 Computer 按 §5 解析注入）：

```python
import os, json, sys

skill_dir = os.environ["TFROBOT_SKILL_DIR"]
session = os.environ["TFROBOT_SESSION_ID"]

with open(f"{skill_dir}/assets/report-template.docx", "rb") as f:
    template = f.read()

# ... 业务逻辑 ...

json.dump(
    {"session": session, "rows_processed": 1234, "output_file": f"{skill_dir}/out/report.docx"},
    sys.stdout,
)
```

> 同一份 SKILL 包既可作为 Robot 配置导入（此时 `scripts/` 不被执行，仅作为 LLM 可读参考），也可由 Computer 侧通过 A2C-SMCP 暴露（此时按 A2C 实施方约定执行）。**协议层不区分**——同一份包，不同挂载方式带来不同的执行能力。

## 10. 不收入项汇总（接入方实施 review checklist）

| 类别 | 不收入项 | 出处 |
| --- | --- | --- |
| Frontmatter 字段 | `arguments` / `argument-hint` / `$ARGUMENTS` / `$N` / `$name`（CLI 残留） | §3.3 / A1 §3.1 |
| | `model` / `effort` / `disable-model-invocation` / `user-invocable` / `agent`（平台决策） | §3.3 / A1 §3.1 |
| | `hooks` / `paths` / `shell`（本地 FS 概念） | §3.3 / A1 §3.1 |
| | `runtime` / `dockerfile` / `image` / `byoi`（执行后端） | §3.3（A3 撤回，见附录 A） |
| | `version` / `secrets` / `network` / `egress` / `visibility` / `audience` / `quota` / `cost` / `signature` / `integrity` | §3.3 / A1 §3.2 |
| 占位符 | `$ARGUMENTS` / `$N` / `$name` | §6.3 |
| | `$TFROBOT_API_ENDPOINT` / `$TFROBOT_AUTH_TOKEN`（无回调通道） | §6.3 |
| | `$TFROBOT_TENANT_ID` / `$TFROBOT_USER_ID`（多租户上下文不暴露） | §6.3 |
| | `$TFROBOT_ARG` / `$TFROBOT_INPUT`（输入语义留给 A2C-SMCP） | §6.3 |
| | `$TFROBOT_EFFORT` / `$TFROBOT_MODEL` / `$TFROBOT_REQUEST_ID` | §6.3 |
| 执行后端 | runtime 枚举 / per-engine Dockerfile / 沙箱 I/O 协议 / 跨 sandbox 通信 / IPC | 由 Computer 侧 A2C-SMCP 自行定义 |
| 协议外能力 | 完整性签名校验、跨租户公共 marketplace、CDN 加速、多 region 分发、Web 编辑器、调试器 UI、资源配额计费、并发隔离精细策略 | 平台产品路线图范畴 |

## 11. 字段集 Freeze 清单

> 本节为协议字段总图，作为接入方实施时的字段对齐基准。任何不在本表中的字段、占位符、文件名平台均不解释。

### 11.1 Frontmatter 字段（6 个）

`name` / `description` / `license` / `compatibility` / `metadata` / `allowed-tools`

### 11.2 受协议解释的特殊文件名（1 个）

`.skillenv`

### 11.3 标准目录名（3 个，非强制）

`scripts/` / `references/` / `assets/`

### 11.4 占位符（3 个）

`$TFROBOT_SKILL_DIR` / `$TFROBOT_SESSION_ID` / `$TFROBOT_ROBOT_ID`

### 11.5 平台内置工具（1 个）

`skills(skill_name, path)`

### 11.6 Marketplace 层级路径（Git 仓库解析）

`<repo>/plugins/<plugin-name>/skills/<skill-name>/SKILL.md` —— 详见独立规范 [Marketplace 规范](../marketplace/protocol.md)

## 附录 A：设计史

A1\~A4 是规范的分步设计文档；本文档是其整合后的权威契约。如本文与 A1\~A4 表述冲突，**以本文为准**（A1\~A4 保留作为决策 rationale 与设计史）。

| 子任务 | 文档 | 最终采纳 |
| --- | --- | --- |
| A1 frontmatter 字段表 | [frontmatter-fields.md](design-history/frontmatter-fields.md) | ✅ 完全对齐 Agent Skills 开放标准；不采纳 Claude Code 扩展字段 |
| A2 `.skillenv` 设计 | [skillenv.md](design-history/skillenv.md) | ✅ 标准 dotenv 双语义；用户 vault 三层强制隔离 |
| A3 `runtime` 枚举 | [runtime-enum.md](design-history/runtime-enum.md) | ❌ **撤回**——A3 曾设计双引擎枚举（`cubesandbox::*` / `e2b::*`）+ BYO Dockerfile；最终决定执行后端归 Computer 侧 A2C-SMCP，`runtime` 字段不入协议 |
| A4 目录 + 占位符 | [directory-placeholders.md](design-history/directory-placeholders.md) | ✅ `TFROBOT_*` 私有命名空间 + `skills` 工具契约 + 3 条安全原则 |

**A3 撤回 rationale**：A3 假设 TFRobot 协议要直接驱动执行后端（双引擎集群 + 镜像构建 + 沙箱注入），因此设计了 `runtime` 字段携带执行模板 + 引擎前缀语义。评审时确认了更清晰的协议分层：
- **Robot 配置侧 SKILL** 本就不执行脚本（仅文档驱动），不需要 `runtime` 字段
- **Computer 侧 SKILL** 的执行后端由 A2C-SMCP 协议（独立规范）表达，不应污染 SKILL 内容契约

因此 `runtime` 字段及其衍生（per-engine Dockerfile / 沙箱 I/O 协议）整体从协议撤出。A3 文档保留作为设计史档案。

**整体设计教训**（A2 / A3 / A4 反溯共识）：

* "复用 X 实现 Y"在跨信任边界场景需先核对 X 的安全前提是否在 Y 中成立（A2 §11）
* 协议层是否绑定具体后端实现 = 协议设计常态张力；最终选择**协议只管内容契约，执行后端归实施层**（A3 撤回教训）
* 私有命名空间 + 跨平台字面透传 > mirror 上游命名（A4 §9 教训）
* 信息边界 vs 能力边界严格分开：URI 可见性 ≠ 越权能力（A4 §9 教训）
