# TFRobotServer SKILL 协议 v1 规范

> Jira：[TFRS-187](https://turingfocus.atlassian.net/browse/TFRS-187)（A5）/ Story [TFRS-180](https://turingfocus.atlassian.net/browse/TFRS-180) / Epic [TFRS-179](https://turingfocus.atlassian.net/browse/TFRS-179)
> 状态：**v1 规范定稿（草稿）** —— A1\~A4 已合并至 develop（[#35](https://cnb.cool/turingfocus/tfrobotv2/TFRobotServer/-/pulls/35) / [#36](https://cnb.cool/turingfocus/tfrobotv2/TFRobotServer/-/pulls/36) / [#37](https://cnb.cool/turingfocus/tfrobotv2/TFRobotServer/-/pulls/37) / [#38](https://cnb.cool/turingfocus/tfrobotv2/TFRobotServer/-/pulls/38)），本文档将它们整合为单一权威契约。
> 范围：协议层（作者可见契约 + 平台必守约束）。具体存储/执行实施由 Module B（[TFRS-181](https://turingfocus.atlassian.net/browse/TFRS-181)）/ Module C（[TFRS-182](https://turingfocus.atlassian.net/browse/TFRS-182)）落地。
> 读者：**主要面向 Module B/C 实施工程师**（作为契约）+ **SKILL 作者**（作为开发指南）。

## 0. 总览

### 0.1 这是什么

TFRobotServer SKILL 协议定义**单个能力包（SKILL）的撰写规范与执行语义**。

**SKILL 是什么**：

* **一个文件夹**，包含一份 `SKILL.md`（frontmatter + body）+ 可选的脚本、参考资料、资源文件
* **平台无关** —— 协议本身**不指定**存储介质与分发渠道；同一个 SKILL 文件夹可在 Claude Code / Cursor / TFRobotServer 等任何兼容客户端加载使用
* TFRobot 数字员工通过加载 SKILL 来扩展能力 —— 与 Anthropic [Agent Skills 开放标准](https://agentskills.io/specification)中的 Skill 概念**等价**

**TFRobotServer 运行时模型差异**：多 Pod K8s 部署 + Sandbox 隔离执行 + 私人凭证 vault 量身设计；具体表现为协议层的 `runtime` 枚举（§6）、`.skillenv` 双语义（§5）、`skills` 工具（§8）等扩展。

### 0.1.1 SKILL 的分发与运行时存储

SKILL 撰写规范本身**不强制**存储介质或分发渠道。在 TFRobotServer 平台内：

| 维度 | 实情 |
| --- | --- |
| **分发渠道（3 种）** | ① 用户在 Portal 直接上传 SKILL 文件夹 ② 用户在 Portal 内主动编辑 ③ 平台从 **Marketplace（Git 仓库）** 拉取 —— 详见独立规范 [D-marketplace-v1.md](../marketplace/protocol-v1.md) |
| **运行时存储** | 所有 SKILL 内容（无论从哪个分发渠道来）**最终落到 MinIO**（Module B [TFRS-181](https://turingfocus.atlassian.net/browse/TFRS-181) 定稿）；FastAPI 主进程不持久化（§10.1） |
| **作者关心什么** | 仅 §2~§10 文件夹内容契约；分发与存储是平台实施细节，作者无需关心 |

### 0.2 与上游标准的关系

* **完全对齐**[Agent Skills 开放标准](https://agentskills.io/specification)的 6 个 frontmatter 字段（`name` / `description` / `license` / `compatibility` / `metadata` / `allowed-tools`），跨 Claude Code / Cursor / Goose 等兼容客户端**互操作不破坏**。
* **唯一独有字段** `runtime` —— Claude Code 不需要（一切在 CLI 主机内执行）；TFRobotServer 需要它指示执行模板与目标引擎。
* **不采纳**任何 Claude Code 私有扩展字段（CLI 残留 / 平台决策项 / 本地 FS 概念），命名空间隔离避免"看似相同实则不同"的隐 bug。

### 0.3 v1 范围与不范围

**v1 收入**：

* Frontmatter 7 字段（6 标准 + 1 独有）+ 标准目录结构 + `.skillenv` 双语义环境变量声明 + per-engine BYO Dockerfile + 引擎前缀化 runtime 枚举 + 3 个运行时占位符 + 平台内置 `skills` 工具 + 纯函数沙箱脚本模型 + 3 条 SKILL 资源访问安全原则。

**v1 不收入**（详见 §12）：

* CLI 残留占位符 / 平台决策性 frontmatter 字段（model / effort / disable-model-invocation 等）/ 完整性签名校验 / 跨租户公共 marketplace / CDN / 多 region 分发 / Web 编辑器 / 调试器 UI / 沙箱脚本 HTTP 回调（TFRS-197 已暂停）/ 资源配额计费 / 并发隔离精细策略。

## 1. 设计原则

v1 设计共 13 条原则。前 10 条来自 A1~A4 的"克制 + 互操作 + 安全"基调；最后 3 条是 SKILL 资源访问安全模型的强制约束。

### 1.1 协议层（10 条）

1. **完全对齐 Agent Skills 开放标准**：6 个 frontmatter 字段全采纳，不破坏跨客户端互操作。
2. **不采纳 Claude Code 扩展字段**：CLI 残留（`$ARGUMENTS` / `arguments` / `argument-hint`）、平台决策项（`model` / `effort` / `disable-model-invocation` / `user-invocable`）、本地 FS 概念（`hooks` / `paths`）等一律不入；理由汇总在 A1 §3。
3. **独有字段仅 1 个**：`runtime` —— 驱动 Module C 选择执行模板与目标引擎。
4. **可由仓库内文件表达的不入 frontmatter**：环境变量 → `.skillenv`；BYO 镜像 → `.<engine>.dockerfile`；frontmatter 仅承载元信息。
5. **占位符私有命名空间**：所有运行时占位符以 `TFROBOT_` 前缀；不 mirror Claude Code `CLAUDE_*` / `$ARGUMENTS`；未识别占位符跨平台加载时字面透传。
6. **`runtime` 取值显式带引擎前缀**：协议层与后端 1:1 对应；平台**同时常驻** CubeSandbox + E2B 双集群，作者按 `runtime` 取值显式路由；**无全局 active-engine 配置**。
7. **强契约（runtime ↔ Dockerfile 文件名一一对应）**：`cubesandbox` ⇔ `.cubesandbox.dockerfile`；`e2b` ⇔ `.e2b.dockerfile`；Tier 0/1 取值禁有 dockerfile；加载器强制校验。
8. **纯函数沙箱脚本模型**：v1 沙箱内脚本**无 host 回调通道** —— 输入走 env / 挂载 / 可选 stdin；输出走 stdout / 文件。需要"读会话 / 调其他工具 / 做对话决策"的场景应写为 `prompt-only` SKILL。与 E2B / CubeSandbox 行业默认一致。
9. **敏感凭证 LLM 不可见**：用户 vault 解析后的明文**只**作为沙箱进程 env 注入，不进入 SKILL.md body 展开路径、不进入 LLM 上下文；A2 §5 三层防护落地。
10. **v1 克制**：所有候选字段 / 占位符 / 取值均需具体 SKILL 场景驱动；不预设未实证的能力。

### 1.2 SKILL 资源访问安全原则（3 条强制）

SKILL 文件存储在 MinIO（B 模块定稿），**作为不可信资产对待**。三条强制约束：

11. **SKILL 资源访问受控**：所有 SKILL 文件访问（LLM 读 `references/` / `scripts/`、sandbox 内 FS 访问）都由平台居间介入；不存在"LLM 直接拿 MinIO 凭据自取"或"FastAPI 把 SKILL 当本地 import / eval"的路径。LLM 端通过平台内置 `skills` 工具（§7）；sandbox 端通过受控 staging 管线挂载（§9）。
12. **FastAPI 主进程不持久化 SKILL**：SKILL 字节从 MinIO 取出后**只**在主进程内存中一次性使用（喂给 LLM 上下文 / staging 给 sandbox），**绝不**写入 Pod FS、**绝不** `subprocess` 执行 SKILL 衍生命令、**绝不**基于 SKILL 文件 FS 权限做决策。SKILL 文件对主进程而言**只是 data**。
13. **Sandbox staging 不经 FastAPI FS**：MinIO → sandbox 卷的文件传递走独立管线（Module B/C 出 ADR），FastAPI 主进程不当中转。**约束**：staging 延迟不淹没 `< 60ms` 沙箱冷启动优势（CubeSandbox 60ms / E2B P95 ~90ms）。

## 2. SKILL 包结构

```
my-skill/                          # 包根目录名 = SKILL.md frontmatter `name`
├── SKILL.md                       # 必需 —— frontmatter + body
├── .skillenv                      # 可选 —— 环境变量声明（A2 / 本文 §5）
├── .cubesandbox.dockerfile        # 条件必需 —— runtime: cubesandbox 时（A3 / 本文 §6）
├── .e2b.dockerfile                # 条件必需 —— runtime: e2b 时
├── scripts/                       # 标准命名 —— 沙箱内执行入口与可执行代码
├── references/                    # 标准命名 —— LLM 渐进式读取的参考文档
├── assets/                        # 标准命名 —— 静态资源
└── <任意其他目录或文件>             # 作者自由扩展，平台不解释
```

### 2.1 强约束

| 项 | 约束 |
| --- | --- |
| 包根目录名 | 必须等于 `SKILL.md` frontmatter 中的 `name` |
| `SKILL.md` | 必须存在于包根；缺失即加载失败 |
| 受协议解释的 hidden-file | 仅 `.skillenv` / `.cubesandbox.dockerfile` / `.e2b.dockerfile` 三个；其他 `.xxx` 平台不解释也不拒绝 |
| `scripts/` / `references/` / `assets/` | 命名标准化但**不强制必填** —— 按需创建即可，缺失加载器不报错 |

### 2.2 语义化命名指引（建议遵循）

| 目录 | 语义 |
| --- | --- |
| `scripts/` | 沙箱内可执行代码（`.py` / `.ts` / `.js` / shell 等） |
| `references/` | LLM 渐进式读取的参考文档（Markdown / 纯文本） |
| `assets/` | 静态资源（图片、字体、文档模板、二进制等） |

## 3. SKILL.md Frontmatter 字段表

YAML frontmatter 位于 SKILL.md 顶部，包裹在 `---` 之间。

### 3.1 必填字段

| 字段 | 类型 | 约束 | 含义 |
| --- | --- | --- | --- |
| `name` | string | 1–64 字符；`[a-z0-9-]`；不以 `-` 开头/结尾；无连续 `--`；**与目录名一致** | SKILL 唯一标识 |
| `description` | string | 1–1024 字符；非空 | "做什么 + 何时用"；Robot 启动时进入 LLM 可见 skill 列表（progressive disclosure 第一层）；**首句包含核心触发关键词** |

### 3.2 可选字段（开放标准）

| 字段 | 类型 | 约束 | 含义 |
| --- | --- | --- | --- |
| `license` | string | 标准未规定字符上限 | 许可声明 |
| `compatibility` | string | 最长 500 字符 | 人类可读环境要求 hint（如"需要互联网访问"、"已预装 PyMuPDF"）；**与 `runtime` 不重叠** |
| `metadata` | map<string,string> | 标准未规定大小上限 | 自由 key-value，跨客户端互操作透传；TFRobotServer **不解释** |
| `allowed-tools` | string \| list | 空格分隔字符串或 YAML 列表 | LLM 推理循环中可直接调用、免授权的工具白名单；**与沙箱执行无关**，仅作用于 LLM-工具循环层面 |

### 3.3 独有字段

| 字段 | 类型 | 默认 | 含义 |
| --- | --- | --- | --- |
| `runtime` | enum（7 个取值，详见 §6） | `prompt-only` | 决定执行模板与目标引擎 |

### 3.4 v1 不引入的字段（完整列表）

CLI 残留：`arguments` / `argument-hint` / `$ARGUMENTS` / `$N` / `$name`
平台决策：`model` / `effort` / `disable-model-invocation` / `user-invocable` / `agent`
本地 FS 概念：`hooks` / `paths`
环境隐式：`shell`（沙箱固定 Linux bash）
候选未引入：`version` / `secrets` / `network` / `egress` / `visibility` / `audience` / `quota` / `cost` / `signature` / `integrity` / `dockerfile` / `image` / `byoi`

理由汇总参见 A1 §3。

## 4. SKILL.md Body 约定

* **格式**：Markdown，由 LLM 直接消费（拼到 prompt 中）
* **相对路径引用**：参照 Agent Skills 开放标准 —— 引用包内文件用"相对 SKILL.md 的路径"（如 `scripts/main.py` / `references/foo.md`）；具体由平台 Read 工具（§7）按"当前激活 SKILL"上下文解析
* **占位符**：SKILL.md body 中的 `$TFROBOT_*` 占位符在激活时由平台展开（§7.2 / §8.1）
* **触发表述**：description 的首句必须包含核心触发关键词；body 可详述步骤、调用工具、引用资源

## 5. `.skillenv` 环境变量声明

仓库根可选文件，声明 SKILL 执行所需的环境变量与密钥来源。**LLM 全程不可见**。

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
| `KEY=VALUE`（VALUE 非空） | **字面量** —— VALUE 整段作为 env var 值传给沙箱；不做任何解析或变量替换 |
| `KEY=`（VALUE 为空） | **用户 vault 引用** —— 以 KEY 为名查询用户私人凭证 vault；查到 → 注入解析后的明文；查不到 → 启动失败 |

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

### 5.4 LLM 不可见性（三层防护，全部实施）

1. **加载层黑名单**（Module B 实施）：SKILL 加载器维护 `_LLM_INVISIBLE_FILES = {".skillenv"}`；LLM 任何路径请求都拒绝返回，记审计日志
2. **沙箱注入而非文件分发**（Module C 实施）：E2B/CubeSandbox 启动器在 `sandbox.create()` 前完成解析，通过 SDK `envs={...}` 注入；**原文件不进入沙箱 FS**
3. **prompt 渲染产物 grep 守护**（CI 黑测试）：断言最终拼到 LLM 的 prompt 字符串中不含 `.skillenv` 任何行内容（KEY 名、VALUE 字面量、注释）

### 5.5 与平台密钥基础设施的边界

| 项 | 边界 |
| --- | --- |
| 协议层语法 | `.skillenv` 没有任何语法能引用 ManagedLLM keystore |
| 解析层映射 | 空 VALUE 时**只**查询用户私人 vault，不查询 ManagedLLM keystore |
| 注入层目标 | 沙箱 env 只含用户 vault 解析结果 + 字面量；**ManagedLLM keystore 内容永不出现在沙箱 env** |
| 允许复用 | TFRSManager 加密传输通道、KeyCache 类基础设施、DI 注入模式 |
| 禁止复用 | ManagedLLM 的 keystore 表、`ManagedKeyClient.get_api_key()` 接口本身、ManagedLLM 已注册的 provider_name 命名空间 |

> 详见 A2 §6。背景：A2 设计稿过程中发现原"复用 ManagedLLM keystore"表述存在信任边界漏洞 —— 一旦把 ManagedLLM keystore 内容注入 `.skillenv`，用户脚本一句 `print(os.environ)` 就能 dump 出平台付费 quota。本协议在协议层、解析层、注入层**三处强制隔离**。

## 6. Runtime 枚举与 Dockerfile 文件

### 6.1 v1 枚举（7 个取值）

平台**同时常驻** CubeSandbox + E2B 双集群；按 `runtime` 取值显式路由（无全局 active-engine 配置）。

| Tier | 取值 | 说明 |
| --- | --- | --- |
| **0 无沙箱** | `prompt-only` | 默认值；纯 prompt + LLM 工具调用编排，无任何脚本执行 |
| **1 平台预制** | `cubesandbox::python` | CubeSandbox 集群 + 平台预制 Python 3.11+ 镜像（含 pandas/numpy 数据栈） |
| | `cubesandbox::node` | CubeSandbox 集群 + 平台预制 Node.js 22+ 镜像（含 tsx/zod/axios/dayjs） |
| | `e2b::python` | E2B 集群 + 平台预制 Python 3.11+ 镜像（同上） |
| | `e2b::node` | E2B 集群 + 平台预制 Node.js 22+ 镜像（同上） |
| **2 BYO** | `cubesandbox` | CubeSandbox 集群 + 作者自带 `.cubesandbox.dockerfile` |
| | `e2b` | E2B 集群 + 作者自带 `.e2b.dockerfile` |

### 6.2 强契约（runtime ↔ Dockerfile 文件名）

| `runtime` 取值 | `.cubesandbox.dockerfile` | `.e2b.dockerfile` | 加载器行为 |
| --- | --- | --- | --- |
| `prompt-only` | 任一存在即拒绝 | 任一存在即拒绝 | 仅两者都不存在时 ✅ |
| `cubesandbox::*` 任一 | 任一存在即拒绝 | 任一存在即拒绝 | 仅两者都不存在时 ✅ |
| `e2b::*` 任一 | 任一存在即拒绝 | 任一存在即拒绝 | 仅两者都不存在时 ✅ |
| `cubesandbox` | **必须存在** | 不允许 | ✅ |
| `e2b` | 不允许 | **必须存在** | ✅ |

错误信息须明确指出违反规则（例："`runtime: cubesandbox` 要求存在 `.cubesandbox.dockerfile`，找到的却是 `.e2b.dockerfile`"）。

### 6.3 各运行时作者可见契约

#### `prompt-only`

| 维度 | 契约 |
| --- | --- |
| 沙箱 | 无；SKILL.md 在 FastAPI 主进程内加载、渲染、拼 prompt |
| `.skillenv` | 不允许存在；存在即报错 |
| 启动开销 | 0 |

#### `cubesandbox::python` / `e2b::python`

| 维度 | 契约 |
| --- | --- |
| 解释器 | Python ≥ 3.11（具体版本由 Module C 定稿） |
| 预装第三方 | `requests` / `httpx` / `pydantic>=2` / `pyyaml` / `python-dateutil` / `pandas` / `numpy` / `matplotlib` / `openpyxl` / `python-docx` / `pillow` |
| 文件系统 | `$TFROBOT_SKILL_DIR` 挂载，可读写；`/tmp` ≥ 1 GB |
| 网络 | 默认允许出站 HTTPS；入站不开放 |
| 资源默认 | 1 vCPU / 1 GB / 8 min wall-clock |

#### `cubesandbox::node` / `e2b::node`

| 维度 | 契约 |
| --- | --- |
| 解释器 | Node.js ≥ 22 LTS |
| TypeScript 运行 | 预装全局 `tsx`；可 `npx tsx scripts/index.ts` 直跑 |
| 预装第三方（全局） | `tsx` / `zod` / `axios` / `dayjs` |
| package.json | 可选；沙箱启动时**不自动** `npm install`（避免冷启动膨胀） |
| 资源默认 | 1 vCPU / 512 MB / 5 min |

#### `cubesandbox` / `e2b`（Tier 2 BYO）

| 维度 | 契约 |
| --- | --- |
| 镜像来源 | 作者 `.<engine>.dockerfile`；平台代为 build & push & 注册模板 |
| 镜像规范 | `cubesandbox` 必须嵌 `envd:49983`（参见 [CubeSandbox BYO 文档](https://github.com/TencentCloud/CubeSandbox/blob/master/docs/guide/tutorials/bring-your-own-image.md)）；`e2b` 必须符合 [E2B Template spec](https://e2b.dev/docs/sandbox-template) |
| 可装内容 | 任意 —— 任何 apt 包、任何语言运行时、专用二进制 |
| `.skillenv` 注入 | 与 Tier 1 一致；同名冲突时 `envs=` 注入覆盖 Dockerfile `ENV` |

### 6.4 引擎路由语义

| `runtime` 取值 | 路由目标 | 镜像来源 | CubeSandbox 挂时 | E2B 挂时 |
| --- | --- | --- | --- | --- |
| `prompt-only` | FastAPI 主进程 | — | ✅ 不受影响 | ✅ 不受影响 |
| `cubesandbox::*` | CubeSandbox 集群 | 平台预制 | ❌ `engine unavailable` | ✅ |
| `e2b::*` | E2B 集群 | 平台预制 | ✅ | ❌ `engine unavailable` |
| `cubesandbox`（BYO） | CubeSandbox 集群 | 本包 `.cubesandbox.dockerfile` 构建产物 | ❌ `engine unavailable` | ✅ |
| `e2b`（BYO） | E2B 集群 | 本包 `.e2b.dockerfile` 构建产物 | ✅ | ❌ `engine unavailable` |

**无全局 active engine 切换**：要换引擎，**作者**改 `runtime` 前缀（BYO 情况下连带改 dockerfile 文件名 + 内容）；不是平台运维改配置。

### 6.5 `.cubesandbox.dockerfile` / `.e2b.dockerfile` 契约

| 项 | 值 |
| --- | --- |
| 位置 | skill 包根目录，与 `SKILL.md` 同级 |
| 是否必填 | 由 §6.2 强契约决定 |
| 文件格式 | 标准 [OCI Dockerfile](https://github.com/opencontainers/image-spec) |
| 构建上下文 | 默认 = skill 包根目录 |
| LLM 可见性 | **不进入黑名单**（与 `.skillenv` 区别）—— Dockerfile 不承载密钥（密钥统一走 `.skillenv`），LLM 可读对 review 有用 |
| 同包两份共存 | **v1 不允许**（违反 §6.2 强契约即报错）；为未来 `runtime` 列表语法预留扩展点 |
| 大小限制 | 建议 ≤ 256 KB Dockerfile + 单次构建产物 ≤ 5 GB（Module C 定稿） |
| 是否进入沙箱 FS | **不进入** —— 仅平台构建期消费的元数据 |

### 6.6 与执行引擎的工程对照

CubeSandbox 与 E2B 在 SDK 与能力面平价（文件 I/O / 进程 / pause-resume / 浏览器 / 网络策略 / RL 均支持）；差异：

| 维度 | CubeSandbox（v1 主选） | E2B（备选） |
| --- | --- | --- |
| 协议 | E2B SDK drop-in | 原生 |
| 隔离 | RustVMM + KVM 独立 guest kernel | gVisor / Firecracker |
| 冷启动 | < 60 ms（P95 90 ms） | 100–300 ms 典型 |
| 单实例开销 | < 5 MB | 数百 MB |
| 国内可用 | 与 TFRobotServer 同区，国内镜像 `cube-sandbox-cn.tencentcloudcr.com` | 出海代理 / 合规自评估 |
| 自托管难度 | 一键脚本 + PVM 兼容普通腾讯云 VM | `e2b-dev/infra` Terraform 仅覆盖 GCP/AWS；2500 GB SSD + 24 CPU 起步 + Cloudflare |

CubeSandbox 是 E2B SDK drop-in，**统一走** `e2b-code-interpreter` SDK；执行器代码一份，仅 `E2B_API_URL` + `template_id` 路由差异。

## 7. 占位符

### 7.1 v1 占位符全集（3 个）

全部以 `TFROBOT_` 前缀；SKILL.md body 中可写 `$TFROBOT_<NAME>` 或 `${TFROBOT_<NAME>}`（等价）。

| 占位符 | Tier 0 `prompt-only` | Tier 1 / Tier 2 沙箱化 |
| --- | --- | --- |
| `$TFROBOT_SKILL_DIR` | 不透明 URI（如 `tfs-skill://<tenant>/<skill>/<version>/`），供 `skills` 工具识别 + 外部 S3 凭据环境拼接使用 | 沙箱本地挂载路径（如 `/skill`） |
| `$TFROBOT_SESSION_ID` | ✅ 当前会话 UUID | ✅ |
| `$TFROBOT_ROBOT_ID` | ✅ 当前 robot 实例 ID | ✅ |

> `$TFROBOT_SKILL_DIR` 的 URI 形式是个**前缀** —— 作者可拼接相对路径（`references/foo.md`）形成完整资源标识；具体 URI 格式由 Module B 定（可能是 `tfs-skill://` / `minio://` / 不透明 token），作者不应解析其内部结构。URI 字面暴露给 LLM **不构成漏洞** —— 安全闸门在 `skills` 工具实现层（§7.3）。

### 7.2 展开规则

**SKILL.md body**：

| 项 | 规则 |
| --- | --- |
| 时机 | 平台在拼到 LLM prompt **之前**完成替换；LLM 见到的是已展开值 |
| 语法 | `$TFROBOT_NAME` 与 `${TFROBOT_NAME}` 等价 |
| 命名空间 | 仅识别 `$TFROBOT_*` 开头；其他 `$VAR` 保持字面原样 |
| 未定义占位符 | `$TFROBOT_FOOBAR` 等 → 加载器报错 |
| runtime 不匹配 | （v1 当前 3 个占位符均跨 runtime 通用；未来若新增 runtime-specific 占位符，违反时报错） |
| 转义 | 不支持；需要字面字符串用 code block 或文本说明 |

**沙箱进程环境变量**：

| 项 | 规则 |
| --- | --- |
| 注入名 | 去掉 `$` / `${}`，直接 env var 名（`TFROBOT_SKILL_DIR` 等） |
| 注入时机 | Module C2（[TFRS-195](https://turingfocus.atlassian.net/browse/TFRS-195)）在 `sandbox.create(envs=...)` 时一次性注入 |
| 与 `.skillenv` 关系 | 共享 env 命名空间但命名避让；`.skillenv` 不允许覆盖平台占位符 |
| 作者读取 | Python `os.environ["TFROBOT_SKILL_DIR"]` / Node `process.env.TFROBOT_SKILL_DIR` / shell `$TFROBOT_SKILL_DIR` |

### 7.3 不收入 v1 的占位符（与不引入理由）

| 候选 | 不收入理由 |
| --- | --- |
| `$ARGUMENTS` / `$N` / `$name` | CLI 残留（A1 §3.1 已排除） |
| `$TFROBOT_API_ENDPOINT` / `$TFROBOT_AUTH_TOKEN` | v1 纯函数沙箱脚本模型（§9）无 host 回调通道；推迟到 v2 |
| `.skillenv` 解析后的用户 vault 明文 | 敏感值隔离 —— 仅作为沙箱 env 注入，不在 body 展开 |
| `$TFROBOT_TENANT_ID` / `$TFROBOT_USER_ID` | 多租户上下文 v1 不暴露；按 tenant 区分应在 robot 配置层 |
| `$TFROBOT_ARG` / `$TFROBOT_INPUT` 类输入占位符 | 输入参数注入语义涉及 LLM 工具调用协议；由 Module C2 实施时定 |
| `$TFROBOT_EFFORT` / `$TFROBOT_MODEL` | 模型与算力档由平台决定，不暴露 |
| `$TFROBOT_REQUEST_ID` | 可观测性走平台日志 / trace |

## 8. 平台内置 `skills` 工具

**LLM 读 SKILL 资源的唯一受控通道**（§1.2 第 11 条）。任何 SKILL 激活期间隐式可用，不需作者写入 `allowed-tools`，也不能被禁用。

### 8.1 工具契约

| 项 | 设计 |
| --- | --- |
| 工具名 | `skills`（与 Claude Code 命名一致） |
| 入参 1 | `skill_name`: string —— 平台已注册 SKILL 的 `name`（§3.1） |
| 入参 2 | `path`: string —— 相对 skill 根的路径（如 `references/foo.md` / `scripts/main.py` / `assets/template.docx`） |
| 返回 | 文件内容（文本直返；二进制走 base64） |
| **黑名单（强制 403）** | `.skillenv` / `.cubesandbox.dockerfile` / `.e2b.dockerfile` / 任何 `.*.dockerfile` 模式 / 任何 `.skillenv*` 模式 |
| **路径校验** | 拒绝 `..` / 绝对路径 / 跳出 skill 根 / 符号链接逃逸 |
| **授权校验** | 当前会话有权访问 `skill_name` 才允许；权限模型由 Module B registry 定 |
| **大小上限** | 建议 ≤ 10 MB（超过返回错误并提示走 sandbox 处理） |
| **作用域** | 仅 LLM 工具调用层；不暴露给沙箱脚本 |
| **跨 SKILL 读** | 天然支持 —— `skill_name` 是显式入参；授权由 registry 把关 |
| **底层依赖** | TFRobotV2 Drive 层「隐身工具 + 动态暴露」能力（[TFROB-477](https://turingfocus.atlassian.net/browse/TFROB-477)） |

### 8.2 典型调用

```python
# LLM 在 prompt-only SKILL 中按 body 指示读取参考文档
skills(skill_name="deploy-helper", path="references/checklist.md")

# 跨 SKILL 引用共享资源（如 shared template / 术语表）
skills(skill_name="common-utils", path="references/glossary.md")

# 黑名单触发拒绝
skills(skill_name="csv-aggregator", path=".skillenv")
# → 403 access denied: protected file
```

## 9. 沙箱脚本 I/O 协议

v1 采用**纯函数模型**（§1.1 第 8 条）；脚本无 host 回调，输入走 env / 挂载 / 可选 stdin，输出走 stdout / 文件。

### 9.1 输入通道

| 通道 | 内容 | 实施 ticket |
| --- | --- | --- |
| 环境变量 | `TFROBOT_*` 占位符（§7）+ `.skillenv` 解析后的明文（§5.2） | [TFRS-195](https://turingfocus.atlassian.net/browse/TFRS-195)（C2） |
| 挂载目录 | SKILL 包目录挂载至 `$TFROBOT_SKILL_DIR`；**MinIO → sandbox 卷的 staging 不经 FastAPI 主进程 FS**（§1.2 第 13 条）；staging 延迟"不淹没 < 60ms 冷启动" | [TFRS-199](https://turingfocus.atlassian.net/browse/TFRS-199)（C6） |
| 启动参数 | LLM tool call 的结构化参数 → 注入方式（argv / stdin JSON / 额外 env）由 C2 定稿 | [TFRS-195](https://turingfocus.atlassian.net/browse/TFRS-195) |

### 9.2 输出通道

| 通道 | 内容 |
| --- | --- |
| stdout | 沙箱进程 stdout 捕获后作为 LLM 见到的 tool result |
| stderr | 落平台日志，作者可查；不进入 LLM 上下文 |
| 产物文件 | 写入 `$TFROBOT_SKILL_DIR/out/` 的文件可选回收（具体策略由 C6 定） |

### 9.3 无 host 回调

脚本**不主动** HTTP 调 TFRobotServer 主进程；无 `$TFROBOT_API_ENDPOINT`。需要"读会话 / 调其他工具 / 做对话决策"的场景应写为 `prompt-only` SKILL，由 LLM 在其工具循环中编排。

**v1 不实现 host 回调**，[TFRS-197](https://turingfocus.atlassian.net/browse/TFRS-197) 暂停推到 v2，等具体业务诉求出现再重启。

## 10. 安全模型

### 10.1 三条强制原则

| 原则 | 实施位置 |
| --- | --- |
| **SKILL 资源访问受控** | LLM 走 `skills` 工具（§8）；sandbox 走平台 staging 挂载（§9.1） |
| **FastAPI 主进程不持久化 SKILL** | 只读字节、不下盘、不 `subprocess` 执行 SKILL 衍生命令、不基于 SKILL 文件 FS 权限做决策；当 data 喂 LLM 一次性使用 |
| **Sandbox staging 不经 FastAPI FS** | MinIO → sandbox 卷独立管线（Module B/C ADR）；延迟不淹没 < 60ms 冷启动 |

### 10.2 LLM 不可见性保障

`.skillenv` 与所有平台敏感值（短期 auth token、用户 vault 明文）必须满足：

* 加载器黑名单拒绝任何 path 请求（§5.4 第 1 条）
* 沙箱注入而非文件分发（§5.4 第 2 条）
* CI grep 守护 prompt 渲染产物（§5.4 第 3 条）

### 10.3 与 ManagedLLM 信任隔离

`.skillenv` 仅服务**用户私人凭证 vault**；ManagedLLM keystore 在协议层、解析层、注入层**三处强制隔离**，物理上无任何语法能引用平台付费 quota（§5.5）。

## 11. 完整示例

### 11.1 极简 `prompt-only`（无脚本、无环境变量、纯参考文档驱动）

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
2. 读取 `references/checklist.md`（描述按环境分组的检查项）
3. 根据当前 robot 状态逐项核对
4. 输出报告，会话 ID 引用 `$TFROBOT_SESSION_ID` 以便后续追溯
```

> 步骤 2 LLM 调用 `skills("deploy-helper", "references/checklist.md")` 取回内容；`$TFROBOT_SESSION_ID` 在激活时展开为具体 UUID。

### 11.2 `cubesandbox::python`（脚本 + 资源 + 用户 vault + 纯函数输出）

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
runtime: cubesandbox::python
compatibility: 已预装 pandas / python-docx；列规则参见 references/column-mapping.md
---

## 执行

1. 沙箱启动后运行 `$TFROBOT_SKILL_DIR/scripts/main.py`
2. 脚本把处理统计 JSON 写到 stdout；LLM 收到 stdout 作为 tool result
```

`.skillenv`：

```dotenv
LOG_LEVEL=INFO
# 注：CSV 抓取本地完成，无需用户 vault；此处仅展示 .skillenv 形式
```

`scripts/main.py`（沙箱内纯函数执行）：

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

> 想跑在 E2B 上：把 `runtime` 改为 `e2b::python`，其他不变。

### 11.3 `cubesandbox` BYO（特定二进制：含 ffmpeg + 私有 wheel）

包目录：

```
video-thumbnail/
├── SKILL.md
├── .cubesandbox.dockerfile
└── scripts/
    └── thumb.py
```

`SKILL.md`：

```yaml
---
name: video-thumbnail
description: 对上传的视频抽取关键帧并生成缩略图。当用户上传视频文件并要求"抽缩略图"时触发。
runtime: cubesandbox
compatibility: 自带镜像包含 ffmpeg 7.x 与私有内部库 turingfocus-media；构建期约 90 秒
---

运行 `$TFROBOT_SKILL_DIR/scripts/thumb.py`；脚本将缩略图 base64 + 元数据 JSON 写到 stdout。
```

`.cubesandbox.dockerfile`：

```dockerfile
FROM ghcr.io/tencentcloud/cubesandbox-base:2026.16

RUN apt-get update \
    && apt-get install -y --no-install-recommends ffmpeg python3 python3-pip \
    && rm -rf /var/lib/apt/lists/*

# 私有 PyPI（构建机能访问）
RUN pip install --index-url https://pypi.turingfocus.internal/simple/ \
        turingfocus-media==1.4.2 pillow
```

> 想跑 E2B：① `runtime` 改 `e2b`；② 文件名改 `.e2b.dockerfile`；③ 内容改符合 [E2B Template spec](https://e2b.dev/docs/sandbox-template)（base image 不同、不需要 envd）。强契约保证两文件不能同包共存。

## 12. v1 不收入项汇总（Module B/C 实施 review checklist）

| 类别 | 不收入项 | 出处 |
| --- | --- | --- |
| Frontmatter 字段 | `arguments` / `argument-hint` / `$ARGUMENTS` / `$N` / `$name`（CLI 残留） | §3.4 / A1 §3.1 |
| | `model` / `effort` / `disable-model-invocation` / `user-invocable` / `agent`（平台决策） | §3.4 / A1 §3.1 |
| | `hooks` / `paths` / `shell`（本地 FS 概念） | §3.4 / A1 §3.1 |
| | `version` / `secrets` / `network` / `egress` / `visibility` / `audience` / `quota` / `cost` / `signature` / `integrity` / `dockerfile` / `image` / `byoi` | §3.4 / A1 §3.2 |
| 占位符 | `$ARGUMENTS` / `$N` / `$name` | §7.3 |
| | `$TFROBOT_API_ENDPOINT` / `$TFROBOT_AUTH_TOKEN`（v1 纯函数模型无回调） | §7.3 |
| | `$TFROBOT_TENANT_ID` / `$TFROBOT_USER_ID`（多租户上下文不暴露） | §7.3 |
| | `$TFROBOT_ARG` / `$TFROBOT_INPUT`（输入语义留给 C2） | §7.3 |
| | `$TFROBOT_EFFORT` / `$TFROBOT_MODEL` / `$TFROBOT_REQUEST_ID` | §7.3 |
| Runtime 枚举 | `shell` / `bash` / `mcp-server`（专用预制不收，由 Tier 2 BYO 承接） | A3 §5 |
| | `python-data` 独立预制（已并入 `python`） | A3 §5 |
| | `python-ml` / `python-pdf` / `python-browser`（巨型镜像，由 Tier 2 BYO 承接） | A3 §5 |
| | `firecracker` / `daytona` / 其他第三方引擎绑定值 | A3 §5 |
| | Tier 1 上的"可选 layered Dockerfile"中间层 | A3 §5 |
| | `runtime: [cube, e2b]` 列表语法（预留未来扩展） | A3 §1 第 8 条 |
| 沙箱脚本能力 | HTTP 回调主进程（[TFRS-197](https://turingfocus.atlassian.net/browse/TFRS-197) 暂停） | §9.3 |
| | 跨 sandbox 通信 / IPC | 不在协议范围 |
| 跨引擎可移植 | Tier 2 BYO 自动跨引擎转译 | A3 §3.4 |
| 协议外能力 | 完整性签名校验、跨租户公共 marketplace、CDN 加速、多 region 分发、Web 编辑器、调试器 UI、资源配额计费、并发隔离精细策略 | Epic [TFRS-179](https://turingfocus.atlassian.net/browse/TFRS-179) |

## 13. 下游 ticket 与跨项目依赖

### 13.1 Module B 实施 ticket

| Ticket | 范围 |
| --- | --- |
| [TFRS-181](https://turingfocus.atlassian.net/browse/TFRS-181)（B 模块）| **SKILL 存储与同步**：Git ↔ MinIO ↔ Postgres 三角架构；MinIO 按 sha256 内容寻址（建议）+ Postgres metadata 索引。<br>**协议级安全实施**：`.skillenv` 加载器黑名单 + grep 守护；per-engine Dockerfile 检测与版本绑定（[TFRS-185](https://turingfocus.atlassian.net/browse/TFRS-185) A3 强契约）。<br>**`skills` 工具实施**（§8）：黑名单 + 路径校验 + registry 授权 + 跨 SKILL 读支持。<br>**SKILL 资源访问安全模型**（§10.1）：FastAPI 不持久化 SKILL、Sandbox staging 不经 FastAPI FS（与 C2/C6 协作）。 |

### 13.2 Module C 实施 ticket

| Ticket | 范围 |
| --- | --- |
| [TFRS-182](https://turingfocus.atlassian.net/browse/TFRS-182)（C Story） | Sandbox 双集群常驻 + 环境变量注入 + 占位符实施总体规划 |
| [TFRS-194](https://turingfocus.atlassian.net/browse/TFRS-194)（C1 模板与构建） | Tier 1 镜像构建（CubeSandbox + E2B 各 2 份，共 4 份）+ Tier 2 BYO 构建管线（Buildkit/kaniko + push + 注册模板）|
| [TFRS-195](https://turingfocus.atlassian.net/browse/TFRS-195)（C2 注入链路） | `.skillenv` 解析 → 沙箱注入；启动参数注入选型（argv / stdin / env）；staging 延迟约束 |
| [TFRS-196](https://turingfocus.atlassian.net/browse/TFRS-196)（C3 用户 vault） | 用户私人凭证 vault 独立设计（与 ManagedLLM 信任隔离） |
| ~~[TFRS-197](https://turingfocus.atlassian.net/browse/TFRS-197)（C4 鉴权通道）~~ | **暂停 v1** —— v1 纯函数模型无 host 回调；推到 v2 |
| [TFRS-198](https://turingfocus.atlassian.net/browse/TFRS-198)（C5 prompt-only 快速路径） | `prompt-only` SKILL 跳过沙箱的快速渲染路径 |
| [TFRS-199](https://turingfocus.atlassian.net/browse/TFRS-199)（C6 占位符 + 挂载） | `$TFROBOT_SKILL_DIR` 等占位符的沙箱内实现；MinIO → sandbox 卷挂载机制（与 B 协作） |

### 13.3 Module D（Marketplace）实施 ticket

| Ticket | 范围 |
| --- | --- |
| [TFRS-201](https://turingfocus.atlassian.net/browse/TFRS-201)（D Story） | Marketplace Git 分发与拉取子系统总体规划 |
| [TFRS-202](https://turingfocus.atlassian.net/browse/TFRS-202)（D1 规范定稿）| Marketplace 协议规范 v1 —— 独立文档 [D-marketplace-v1.md](../marketplace/protocol-v1.md)（本 PR 一并交付）|
| [TFRS-203](https://turingfocus.atlassian.net/browse/TFRS-203)（D2 schema） | `MarketplaceSource` 5 种类型 + Postgres schema |
| [TFRS-204](https://turingfocus.atlassian.net/browse/TFRS-204)（D3 Loader） | Loader Strategy 接口 + GitLoader / UploadLoader / InlineEditLoader |
| [TFRS-205](https://turingfocus.atlassian.net/browse/TFRS-205)（D4 Reconciler） | 双层状态 Reconciler（Postgres intended + MinIO materialized + Celery beat） |
| [TFRS-206](https://turingfocus.atlassian.net/browse/TFRS-206)（D5 API） | `MarketplaceService` 函数式 API + FastAPI 端点 |
| [TFRS-207](https://turingfocus.atlassian.net/browse/TFRS-207)（D6 三态协作） | 与 Robot/Factory 三态生命周期协作（Template/Online/Draft + 级联策略） |
| [TFRS-208](https://turingfocus.atlassian.net/browse/TFRS-208)（D7 端到端） | 端到端跑通 + 至少一个示例 Marketplace 仓库 |

### 13.4 跨项目依赖

| 项目 | Ticket | 范围 | 阻塞关系 |
| --- | --- | --- | --- |
| TFRobotV2 (TFROB) | [TFROB-477](https://turingfocus.atlassian.net/browse/TFROB-477) | Drive 层「隐身工具 + 动态暴露」能力 —— `skills` 工具 / 未来 `SkillSandbox` 的协议级支持 | **不阻塞** A1\~A5 文档定稿；**阻塞** Module C 端到端实施时（C1 落地 SkillSandbox 工具时） |

## 14. v1 字段集 Freeze 清单

> 本节为 v1 协议字段总图，作为 Module B/C/D 实施时的字段对齐基准。任何不在本表中的字段、占位符、文件名平台均不解释。

### 14.1 Frontmatter 字段（7 个）

`name` / `description` / `license` / `compatibility` / `metadata` / `allowed-tools` / `runtime`

### 14.2 Runtime 取值（7 个）

`prompt-only` / `cubesandbox::python` / `cubesandbox::node` / `e2b::python` / `e2b::node` / `cubesandbox` / `e2b`

### 14.3 受协议解释的特殊文件名（3 个）

`.skillenv` / `.cubesandbox.dockerfile` / `.e2b.dockerfile`

### 14.4 标准目录名（3 个，非强制）

`scripts/` / `references/` / `assets/`

### 14.5 占位符（3 个）

`$TFROBOT_SKILL_DIR` / `$TFROBOT_SESSION_ID` / `$TFROBOT_ROBOT_ID`

### 14.6 平台内置工具（1 个）

`skills(skill_name, path)`

### 14.7 Marketplace 层级路径（Git 仓库解析）

`<repo>/plugins/<plugin-name>/skills/<skill-name>/SKILL.md` —— 详见独立规范 [D-marketplace-v1.md](../marketplace/protocol-v1.md)

## 附录 A：设计史

A1\~A4 是 v1 规范的分步设计文档；本文档是其整合后的权威契约。如本文与 A1\~A4 表述冲突，**以本文为准**（A1\~A4 保留作为决策 rationale 与设计史）。

| 子任务 | Jira | 文档 | PR | 关键产出 |
| --- | --- | --- | --- | --- |
| A1 frontmatter 字段表 | [TFRS-183](https://turingfocus.atlassian.net/browse/TFRS-183) | [A1-frontmatter-fields.md](design-history/frontmatter-fields.md) | [#35](https://cnb.cool/turingfocus/tfrobotv2/TFRobotServer/-/pulls/35) | 完全对齐 Agent Skills 开放标准 + 1 独有字段 `runtime`；不采纳 Claude Code 扩展字段 |
| A2 `.skillenv` 设计 | [TFRS-184](https://turingfocus.atlassian.net/browse/TFRS-184) | [A2-skillenv.md](design-history/skillenv.md) | [#36](https://cnb.cool/turingfocus/tfrobotv2/TFRobotServer/-/pulls/36) | 标准 dotenv 双语义；ManagedLLM keystore 三层强制隔离；A2 反溯拨乱"复用 ManagedLLM"信任边界漏洞 |
| A3 runtime 枚举 | [TFRS-185](https://turingfocus.atlassian.net/browse/TFRS-185) | [A3-runtime-enum.md](design-history/runtime-enum.md) | [#37](https://cnb.cool/turingfocus/tfrobotv2/TFRobotServer/-/pulls/37) | 双引擎并存 + 引擎前缀语法 + BYO Dockerfile；推翻"E2B 模板对齐"框架；撤回 TFRS-194 原"作者不能自带 Dockerfile" |
| A4 目录 + 占位符 | [TFRS-186](https://turingfocus.atlassian.net/browse/TFRS-186) | [A4-directory-placeholders.md](design-history/directory-placeholders.md) | [#38](https://cnb.cool/turingfocus/tfrobotv2/TFRobotServer/-/pulls/38) | `TFROBOT_*` 私有命名空间 + 纯函数沙箱模型 + `skills` 工具契约 + 3 条安全原则；推翻 HTTP 回调设计，TFRS-197 暂停 v1 |

**整体设计教训**（A2 / A3 / A4 反溯共识）：

* "复用 X 实现 Y"在跨信任边界场景需先核对 X 的安全前提是否在 Y 中成立（A2 §11）
* 协议层是否绑定具体后端实现 = 协议设计常态张力；本协议选「显式 > 暗中切换」（A3 §9.2）
* 私有命名空间 + 跨平台字面透传 > mirror 上游命名（A4 §9 教训）
* 信息边界 vs 能力边界严格分开：URI 可见性 ≠ 越权能力（A4 §9 教训）
* 沙箱化是为隔离，脚本不应回头调度平台（A4 §9 架构教训）

## 准出对照

完成标准（来自 [TFRS-187](https://turingfocus.atlassian.net/browse/TFRS-187) 任务描述）：

* [x] 文档结构覆盖：协议概述（§0）、frontmatter（§3）、目录（§2）、`.skillenv`（§5）、`.cubesandbox.dockerfile` / `.e2b.dockerfile`（§6.5）、占位符（§7）、runtime 枚举（§6）、`skills` 工具（§8）、示例 SKILL（§11）
* [x] 至少 2 个完整示例 SKILL —— §11 提供 3 个（prompt-only / `cubesandbox::python` / `cubesandbox` BYO）
* [x] v1 字段集 freeze —— §14 freeze 清单 + §12 不收入项汇总
* [x] 三条安全原则（§1.2 / §10.1）与纯函数沙箱模型（§9）在总规范中明确表达
* [x] Marketplace 分发标准 —— 独立规范 [D-marketplace-v1.md](../marketplace/protocol-v1.md)（A5 仅 §0.1.1 简引）
