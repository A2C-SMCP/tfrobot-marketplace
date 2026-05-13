# A1 — SKILL.md frontmatter 字段表与 Claude Code 子集裁定

> Jira：[TFRS-183](https://turingfocus.atlassian.net/browse/TFRS-183)（Story [TFRS-180](https://turingfocus.atlassian.net/browse/TFRS-180) / Epic [TFRS-179](https://turingfocus.atlassian.net/browse/TFRS-179)）
> 范围：仅 frontmatter；`.skillenv`、`runtime` 枚举值、目录结构与占位符分别在 A2 / A3 / A4 中定稿。
> 结论一句话：**完全对齐 [Agent Skills 开放标准](https://agentskills.io/specification) + 1 个独有字段 `runtime`，不采纳任何 Claude Code 扩展字段**。

## 1. 裁定原则

1. **完全对齐 Agent Skills 开放标准** ([agentskills.io](https://agentskills.io/specification))：作者写一份 SKILL.md，可在 Claude Code / Cursor / Goose 等任何兼容客户端复用；TFRobotServer 不破坏互操作性，也不自创字段（除一个必要扩展）。开放标准的全部 6 个字段（`name` / `description` / `license` / `compatibility` / `metadata` / `allowed-tools`）一律采纳。
2. **不采纳 Claude Code 扩展字段**：`when_to_use` / `arguments` 三件套 / `model` / `effort` / `context` / `agent` / `hooks` / `paths` / `shell` / `disable-model-invocation` / `user-invocable` 一律不入。理由汇总见 §3.1，核心是这些字段要么是 CLI 残留（Claude Code 文档明确写「Custom commands have been merged into skills」），要么是平台决策（模型选择、调用门禁、可见性）——后者应由平台元数据层（模块 B Postgres registry）/ robot 配置承载，不由 Skill 作者单方面声明。
3. **我方独有字段控制在 1 个**：仅 `runtime`（执行容器选择，A3 定枚举）。Claude Code 不需要此字段（一切跑在 CLI 主机），TFRobotServer 需要它驱动模块 C 选择 E2B 模板。
4. **可由仓库内文件表达的不入 frontmatter**：密钥/环境变量走 `.skillenv`（A2），目录结构与占位符走约定（A4）。
5. **第一版克制**：签名校验、可见性矩阵、网络策略、配额、跨租户 marketplace、版本号等候选字段一律不入 v1（理由见 §3.2）。

## 2. v1 字段表

### 2.1 必填字段（开放标准）

| 字段 | 类型 | 约束 / 默认 | 含义 |
| --- | --- | --- | --- |
| `name` | string | 必填；1–64 字符；`[a-z0-9-]`；不以 `-` 开头/结尾；无连续 `--`；**与目录名一致** | Skill 唯一标识。Robot 通过此名加载/调用。 |
| `description` | string | 必填；1–1024 字符；非空 | 描述「做什么 + 何时用」。Robot 启动时仅加载 description 进入 LLM 可见 skill 列表（progressive disclosure 第一层）。**首句务必包含核心触发关键词**。 |

### 2.2 可选字段（开放标准）

> 必填性、字符上限、命名约束遵循 [agentskills.io/specification](https://agentskills.io/specification) §3 Frontmatter 表；下方「约束 / 默认」列**加粗**项为标准强制约束。

| 字段 | 类型 | 约束 / 默认 | 含义 / 在 TFRobotServer 的解释 |
| --- | --- | --- | --- |
| `license` | string | 可选；标准未规定字符上限 | 许可声明。可填许可证名（如 `Apache-2.0`、`Proprietary`）或仓库内许可文件引用。v1 以企业内部使用为主，作者可空着；保留字段为后续跨租户/外部发布留接口。 |
| `compatibility` | string | 可选；**最长 500 字符** | 自由文本，描述 Skill 的环境/工具要求（如「需要互联网访问」「需要 Python 3.14+」「需要 `kubectl` 上下文」）。**与 `runtime` 不重叠**：`runtime` 是机器可读的执行容器枚举，`compatibility` 是人类可读的额外说明（给作者/审核者看的 README hint），二者各司其职。 |
| `metadata` | map<string,string> | 可选；默认 `{}`；标准未规定大小上限 | 自由 key-value，供跨客户端互操作时承载额外属性（如 Claude Code 私有字段、其他 Agent 客户端的私有字段）。**TFRobotServer 自身不解释此字段，仅透传保留**。平台所需结构化元数据（作者、版本、权限、配额、可见性等）走 Postgres registry，不通过 metadata 反向覆盖。 |
| `allowed-tools` | string \| list | 可选；标准未规定长度上限（**Experimental**） | Skill 激活期间 **LLM 推理循环中**可直接调用、免授权的工具白名单。格式：空格分隔字符串或 YAML 列表（与开放标准、Claude Code 一致）。列表元素是 TFRobot drive/tool 注册名或 MCP tool 名（如 `drive.tool.shell` / `mcp.cnb.cnb_get_issue`）；写法属本地约定，跨客户端加载时各自按本地工具空间解析。细粒度许可语法（`tool(arg:*)`）的具体语义由 A5 / 模块 C 敲定。 |

### 2.3 我方独有字段

| 字段 | 类型 | 默认 | 含义 |
| --- | --- | --- | --- |
| `runtime` | enum | `prompt-only` | 决定执行容器/沙箱模板。**枚举值集合由 [A3](runtime-enum.md)（[TFRS-185](https://turingfocus.atlassian.net/browse/TFRS-185)）定稿为 7 值，全部沙箱化取值显式带引擎前缀**：Tier 0 `prompt-only`（无沙箱）；Tier 1 平台预制 4 个 `<engine>::<lang>`（`cubesandbox::python` / `cubesandbox::node` / `e2b::python` / `e2b::node`）；Tier 2 引擎绑定 BYO 2 个 `<engine>`（`cubesandbox` 需配 `.cubesandbox.dockerfile`；`e2b` 需配 `.e2b.dockerfile`）。平台同时常驻 CubeSandbox + E2B 两套集群，按 `runtime` 取值路由；作者按需选择。 |

## 3. v1 不引入的字段及理由

### 3.1 Claude Code 扩展字段不引入

| 字段 | 不引入理由 |
| --- | --- |
| `when_to_use` | 与 `description` 无实质语义差异 —— Claude Code 自身就是把它拼接在 `description` 后参与同一份 skill listing、共享同一个 1536 字符上限，没有独立代码路径；开放标准也未收录此字段。两个字段反而让作者陷入「关键词该写哪里」的分裂。v1 统一让 `description` 同时承载「做什么 + 何时用」（首句做什么，后续句子讲触发条件）。 |
| `arguments` / `argument-hint` / `$ARGUMENTS` / `$N` / `$name` | **CLI 残留，整组移除。** Claude Code 文档明确写「Custom commands have been merged into skills」——这套位置参数声明 + 占位符替换是旧 slash command 合并进 skill 后的历史包袱，本质是「用户在 CLI 敲 `/skill foo bar` 时按位置切参数」的机制。TFRobotServer 入口是 Robot Chat（自然语言）/ API（结构化 payload）/ Portal UI，三者都没有 CLI 菜单概念：LLM 激活时直接从会话上下文读取所需信息，用户显式触发也是说自然语言、由 LLM 解析意图。环境型占位符（运行时上下文注入，如 session id）是否引入、如何命名，由 A4 (TFRS-186) 单独评估（见 §4.2）。 |
| `disable-model-invocation` | 「LLM 能否自主激活某 Skill」是**平台调用门禁**问题，应由 robot 配置 / Postgres registry 决定哪些 Skill 在当前会话中暴露给 LLM，而非由 Skill 作者在 frontmatter 单方面声明（同一 Skill 在不同 robot/租户/会话中可能策略不同）。带副作用的动作可在 Skill body 内显式要求「执行前向用户确认目标」。 |
| `user-invocable` | 同上 —— Skill 对用户的可见性属授权决策，由 Portal/registry 控制，不由 Skill 作者声明。 |
| `model` / `effort` | 模型与算力档由平台按租户/计费策略决定，作者不应在 Skill 内绕过；属于平台元数据层而非 SKILL.md。 |
| `context: fork` + `agent` | TFRobotServer 没有 Claude Code 的 subagent fork 概念；多会话/线程边界由 Robot 层管理，Skill 不应感知。 |
| `hooks` | Claude Code hooks 依赖本地事件循环（PreToolUse / Stop 等）；TFRobotServer 用 TFRobot 框架原生扩展点（callbacks / extensions），不重复造一套 frontmatter hook DSL。 |
| `paths` | Claude Code 按「当前编辑文件 glob」激活 skill；TFRobotServer 是 Agent-Computer 分离架构，没有"当前编辑文件"概念，激活由 robot 上下文（系统 prompt / 工具调用流）决定。 |
| `shell` | 沙箱固定 Linux，强制 bash；不需要 PowerShell 选项。 |

### 3.2 候选字段中刻意不引入

| 字段 | 不引入理由 |
| --- | --- |
| `version` | Git commit hash + MinIO 对象版本天然承载版本信息（模块 B），frontmatter 内人工填写的版本号必然漂移，反成负担。若需对外暴露版本号，可由 registry 派生展示。 |
| `secrets` | 密钥引用统一走仓库根的 `.skillenv` 文件（A2 定稿），且 **LLM 全程不可见**——写进 frontmatter 会污染 description 同样会被 LLM 加载的上下文，违反隔离前提。 |
| `network` / `egress` | 网络策略由 `runtime` 选中的执行模板隐式决定（取值显式声明目标引擎，平台按取值路由；Tier 2 BYO 时由作者在 `.<engine>.dockerfile` 中自决），不暴露给 Skill 作者在 frontmatter 中声明。 |
| `visibility` / `audience` | 可见性矩阵（租户、组织、用户）属平台层授权决策，由 Portal 配置写入 Postgres registry，Skill 作者无权声明。 |
| `quota` / `cost` | 资源配额与计费策略 v1 不做精细化（Epic 范围外明确列出），暂不需要字段。 |
| `signature` / `integrity` | 完整性签名 v1 范围外（Epic 明确列入「v1 不做」）。 |
| `dockerfile` / `image` / `byoi` | BYO 自定义镜像通过 `runtime ∈ {cubesandbox, e2b}` + skill 包根对应文件 `.cubesandbox.dockerfile` 或 `.e2b.dockerfile`（与 runtime 取值严格对应）表达（详见 [A3](runtime-enum.md) §2.3 / §6.4），**不在 frontmatter 内**。理由：Dockerfile 是多行文件，强行塞进 YAML frontmatter 既不直观也不可扩展；保持「frontmatter 表达元信息 + 仓库内文件表达多行/结构化内容」的分工，与 `.skillenv` 一致。 |

## 4. 与 Claude Code 的差异点（实施侧）

> 采纳的全部是开放标准 + `runtime`，无 Claude Code 同名扩展字段重叠；以下列实施侧需要在模块 B/C 落地时对齐的几点。

### 4.1 `allowed-tools`：工具书写约定差异（仅风格层面）

**字段语义两边一致**：声明 Skill 激活期间 LLM 推理循环中可直接调用、免授权的工具白名单。**与 E2B 沙箱执行无关** —— E2B 由 `runtime` 字段驱动，承载脚本/进程级执行；`allowed-tools` 完全在 LLM-工具循环层面起作用，两者正交。

**格式一致**：开放标准 / Claude Code / TFRobotServer 三者都接受**空格分隔字符串或 YAML 列表**。SKILL.md 在不同客户端加载时只按本地工具空间解析其中的字符串。

**工具书写风格不同（仅约定，非 schema 差异）**：

* Claude Code：`Bash` / `Read` / `Grep` 等本地工具名，可带细粒度许可如 `Bash(git add *)`。
* TFRobotServer：TFRobot drive 注册名 / MCP tool 名，如 `drive.tool.shell` / `mcp.cnb.cnb_get_issue`，沿用细粒度许可写法形如 `drive.tool.shell(git:*)`。

由于 schema 上只是空格分隔字符串，跨客户端互操作不被破坏 —— 平台解析时各自按本地工具空间识别，未识别的工具名通常忽略或在审核期提示。细粒度许可语法（`tool(arg:*)`）的具体语义由 A5 / 模块 C 评审定稿。

### 4.2 占位符替换机制：开放标准未定义，A4 评估是否引入

**开放标准 [agentskills.io/specification](https://agentskills.io/specification) 不定义任何字符串占位符替换机制**。它对 Skill body 引用文件的全部约定是一句话：「use relative paths from the skill root」（例 `scripts/extract.py`），由 agent runtime 自行解析路径。Claude Code 的 `${CLAUDE_SKILL_DIR}` / `${CLAUDE_SESSION_ID}` / `${CLAUDE_EFFORT}` 与 `$ARGUMENTS` 系列都是 **Claude Code 私有扩展**，不是标准约束。

TFRobotServer 在此层与 Claude Code 的差异比「换前缀」更深：

* **「目录」概念不直接成立** —— Skill 存储在 MinIO（对象存储）而非 Pod 本地 FS；运行时由模块 C 按需把 Skill 包投递到 E2B 沙箱或 prompt 上下文。「skill 根」本质是一个**逻辑路径前缀**或**对象引用**，不是 Pod-local 的目录。
* **是否需要把运行时上下文（如 session id、effort 级别等）以占位符形式注入 Skill body**，还是让 runtime 在调用前直接拼到 prompt 里，是另一个独立决策。

**A1 不做决定**，仅明确：

* 开放标准没有占位符机制，不引入也不破坏互操作。
* 不机械 mirror Claude Code 的 `${CLAUDE_*}` 系列。
* 参数型占位符（`$ARGUMENTS` / `$N` / `$name`）已在 §3.1 整组排除（CLI 残留）。
* 是否引入任何环境/运行时占位符、引入哪些、命名规则 → **A4 (TFRS-186) 定稿**。
* 在 A4 给出方案前，Skill 作者用 **相对 SKILL.md 的路径**引用同包资源（如 `scripts/foo.py`、`references/bar.md`），与开放标准一致。

### 4.3 `runtime`：独有字段，决定执行容器与目标引擎

Claude Code 不需要此字段（一切在 CLI 主机进程内执行）。TFRobotServer 需要 `runtime` **取值显式声明目标引擎与模板**（平台同时常驻 CubeSandbox + E2B 两套集群，按取值路由；无全局 active-engine 配置）：

* `prompt-only`（默认）→ 无沙箱，纯 prompt + tool_calls 编排路径
* `cubesandbox::python` / `cubesandbox::node` / `e2b::python` / `e2b::node`（Tier 1）→ 路由到对应引擎集群 + 平台预制的 lang stack
* `cubesandbox` / `e2b`（Tier 2 BYO）→ 路由到对应引擎集群 + 平台为该 skill 用 `.cubesandbox.dockerfile` / `.e2b.dockerfile` 代构建的自定义镜像

完整契约、强契约规则（runtime 取值 ↔ 文件名一一对应）、跨引擎路由语义见 [A3](runtime-enum.md)。

## 5. 完整示例

### 5.1 极简（prompt-only，纯指令）

```yaml
---
name: summarize-uncommitted-changes
description: 总结当前 robot 工作目录下的未提交改动，标记风险点。当用户问「我改了什么」或要求 review diff 时触发。
---

## 任务
基于工具调用拉取 `git diff HEAD`，用 2–3 个 bullet 总结，列出风险点……
```

### 5.2 标注 license + compatibility + allowed-tools 的副作用动作

```yaml
---
name: deploy
description: 把当前 robot 配置发布到目标环境（dev / staging / prod）。当用户明确说「发布」「上线」「deploy」时触发。执行任何写动作前必须向用户确认目标环境与版本号。
license: Proprietary (TFRobotServer Internal)
compatibility: 需要 kubectl 上下文与 CNB 构建权限；目标环境名必须由用户在会话中提供。
allowed-tools: Base Read Grep
---

根据用户在会话中指定的目标环境执行部署：
1. 解析目标环境（如未指定，反问用户）
2. 向用户复述目标环境 + 版本号并请求确认
3. 调用 CNB 构建工具触发构建
4. 等待构建结果并报告……
```

> Skill 的「LLM 是否可自主激活」由 robot 配置/registry 控制（见 §3.1 `disable-model-invocation` 不引入理由）；副作用动作的安全门禁通过 description 中的「触发时机」描述 + body 中的「执行前向用户确认」步骤双重表达。`allowed-tools` 只声明本 Skill 激活期间所需的工具集合，不替代调用门禁。

### 5.3 sandboxed runtime

```yaml
---
name: pdf-extract
description: 抽取 PDF 中的表格与正文文本。当用户上传 PDF 或要求解析 PDF 时触发。
runtime: cubesandbox::python
compatibility: 依赖 PyMuPDF 与 pdfplumber（已预装在 `cubesandbox::python` 模板中）。
---

运行 `scripts/extract.py`（相对 SKILL.md 的路径，由 runtime 解析），对会话中给出的 PDF 路径执行抽取……
```

### 5.4 跨客户端互操作（metadata 透传）

```yaml
---
name: code-review-helper
description: 按本组架构规范审查代码变更。当用户要求 review PR / diff / 变更时触发。
metadata:
  author: platform-team
  homepage: https://wiki.internal/skills/code-review-helper
---

……
```

> `metadata` 字段允许作者附加平台不解释的自由属性（用于跨 Agent 客户端展示），TFRobotServer 仅透传保留，不据此做权限/路由决策。

## 6. 准出对照

完成标准（来自 TFRS-183 任务描述）：

* [x] frontmatter 字段表（必填/可选、类型、默认值、含义）— §2
* [x] 明确不引入的字段及理由 — §3
* [x] 与 Claude Code 字段语义差异点列表 — §4

依赖下游子任务（A1 已声明、由 A2/A3/A4/A5 定稿）：

* `runtime` 字段的具体枚举值与对应 E2B 模板 → A3 (TFRS-185)
* `.skillenv` 文件格式与密钥语法 → A2 (TFRS-184)
* 是否引入任何占位符替换机制（开放标准未定义；MinIO 存储 + E2B 执行模型与 Claude Code 本地 FS 不同）、引入哪些、命名规则 → A4 (TFRS-186)
* `allowed-tools` 工具命名空间映射与细粒度许可语法（`tool(arg:*)`）敲定 → A5 (TFRS-187) 整合评审
