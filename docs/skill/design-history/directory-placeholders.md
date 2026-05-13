# A4 — SKILL 包目录结构与占位符约定

> Jira：[TFRS-186](https://turingfocus.atlassian.net/browse/TFRS-186)（Story [TFRS-180](https://turingfocus.atlassian.net/browse/TFRS-180) / Epic [TFRS-179](https://turingfocus.atlassian.net/browse/TFRS-179)）
> 范围：仅 SKILL 包内**目录命名约定**与 SKILL.md / 脚本中可见的**运行时占位符**集合及展开规则。具体挂载链路 / 占位符注入机制由模块 C（[TFRS-199](https://turingfocus.atlassian.net/browse/TFRS-199) C6）实施；输入参数注入语义由 [TFRS-197](https://turingfocus.atlassian.net/browse/TFRS-197)（C4）定。
> 结论一句话：**目录结构沿用 Agent Skills 标准 + 3 个语义化命名目录（`scripts/` / `references/` / `assets/`）+ 3 个特殊文件（`.skillenv` / `.cubesandbox.dockerfile` / `.e2b.dockerfile`），其余开放；运行时占位符 v1 收 3 个 `$TFROBOT_*`，全部带 `TFROBOT_` 前缀，激活时在 SKILL.md body 中展开、同名以环境变量形式注入沙箱进程；v1 采用**纯函数沙箱脚本模型**（脚本无 host 回调，输入走 env / 挂载 / stdin、输出走 stdout / 文件），与 E2B / CubeSandbox 行业默认一致；CLI 残留占位符（`$ARGUMENTS` / `$N` / `$name`）与 host 回调相关占位符（`$TFROBOT_API_ENDPOINT`）均不收入 v1**。

## 1. 裁定原则

1. **目录半开放**：必须有 `SKILL.md`；`scripts/` / `references/` / `assets/` 为标准语义化命名（作者按用途归类）；其余目录作者可任意添加，平台不强制闭集。
2. **特殊文件名占位**：3 个 hidden-file 前缀的根级文件由前序子任务定稿：`.skillenv`（A2）、`.cubesandbox.dockerfile` / `.e2b.dockerfile`（A3）—— 本文档统一汇总命名空间。
3. **占位符引擎中立 + 私有命名空间**：所有 v1 占位符均以 `TFROBOT_` 前缀，避免与 OS / SDK / 用户脚本环境变量冲突，也不与 Claude Code 的 `CLAUDE_*` / `$ARGUMENTS` 系列冲突或试图 mirror。
4. **同名跨形态统一**：每个占位符在 SKILL.md body 中和沙箱进程环境变量中**同名同语义**；作者一处习得、两处可用。
5. **激活时 body 展开**：平台在把 SKILL.md body 拼到 LLM prompt **之前**完成 `$VAR` / `${VAR}` 替换，LLM 见到的是已展开的真实值（与 Claude Code 一致）。
6. **`$TFROBOT_SKILL_DIR` 跨 runtime 语义统一**：同一占位符名，展开值随 runtime 形态变化 —— 沙箱化 runtime 下展开为本地挂载路径；`prompt-only` 下展开为平台内部不透明 URI（详见 §3.1）。**作者只需记一个名字**。
7. **不收 CLI 残留**：`$ARGUMENTS` / `$N` / `$name` 等位置参数型占位符**已由 A1 §3.1 整组排除**（属"Custom commands have been merged into skills"的历史包袱），本文档保持一致，不重提引入。
8. **不暴露敏感值到 body**：用户 vault 解析后的明文等敏感凭证**只**通过沙箱进程 env 注入，**不**进入 v1 占位符集合、**不**在 SKILL.md body 中展开（与 A2 §5 LLM 不可见性三层防护一致）。
9. **纯函数沙箱脚本模型**：v1 沙箱内脚本无 host 回调通道 —— 输入走 env / 挂载 / 可选 stdin、输出走 stdout / 文件。脚本不主动调用 TFRobotServer 内部 API；需要"读会话 / 调其他工具 / 做对话决策"的场景应写为 `prompt-only` SKILL，让 LLM 在其工具循环中编排。与 E2B / CubeSandbox 行业默认一致（详见 §6.4 与 §9）。
10. **v1 克制**：3 个占位符覆盖「定位资源 + 标识会话 + 标识 robot」最小集；新增项必须有具体 SKILL 场景驱动。

### 1.1 SKILL 资源访问安全原则（新增）

SKILL 文件存储在 MinIO（B 模块定稿），作为**不可信资产**对待。三条强制约束：

11. **SKILL 资源访问受控**：所有 SKILL 文件访问（LLM 读 `references/` / `scripts/`、sandbox 内 FS 访问）都由平台居间介入；不存在"LLM 直接拿 MinIO 凭据自取"或"FastAPI 把 SKILL 当本地 import / eval"的路径。LLM 端通过平台内置 `skills` 工具读资源（§6.5）；sandbox 端通过受控 staging 管线挂载（§6.4）。
12. **FastAPI 主进程不持久化 SKILL**：SKILL 字节从 MinIO 取出后**只**在主进程内存中存活一次性使用（喂给 LLM 上下文 / staging 给 sandbox），**绝不**写入 Pod FS、**绝不** `subprocess` 执行任何 SKILL 衍生命令、**绝不**基于 SKILL 文件 FS 权限做决策。SKILL 文件内容对主进程而言**只是 data**。
13. **Sandbox staging 不经 FastAPI 文件系统**：MinIO → sandbox 卷的文件传递走独立管线（具体方案由 C2/C6 出 ADR，详见 §6.4 / §9.5），FastAPI 主进程不当中转。staging 延迟约束：**不能淹没 < 60ms 沙箱冷启动优势**（CubeSandbox 60ms / E2B P95 ~90ms）；具体实现方向（per-skill-version baked image / lazy FUSE mount / sandbox 直拉 + signed URL / speculative pre-warm 等）由 C 模块按 SKILL 体积分布选型。

## 2. 标准目录结构

```
my-skill/                          # skill 包根（目录名 = SKILL.md frontmatter `name`）
├── SKILL.md                       # 必需。frontmatter + body
├── .skillenv                      # 可选。环境变量声明（A2 定稿）
├── .cubesandbox.dockerfile        # 条件必需 —— runtime: cubesandbox 时（A3 定稿）
├── .e2b.dockerfile                # 条件必需 —— runtime: e2b 时（A3 定稿）
├── scripts/                       # 标准命名 —— 沙箱内执行入口与可执行代码
│   ├── main.py
│   ├── lib/
│   └── ...
├── references/                    # 标准命名 —— LLM 渐进式读取的参考文档
│   ├── architecture.md
│   ├── checklist.md
│   └── ...
├── assets/                        # 标准命名 —— 静态资源（图片、模板、字体等）
│   ├── logo.png
│   └── template.docx
└── <任意其他目录或文件>             # 作者自由扩展，平台不解释
```

### 2.1 强约束

| 项 | 约束 |
| --- | --- |
| 包根目录名 | 必须等于 `SKILL.md` frontmatter 中的 `name`（A1 §2.1） |
| `SKILL.md` | 必须存在于包根；缺失即加载失败 |
| 隐藏文件 | `.skillenv` / `.cubesandbox.dockerfile` / `.e2b.dockerfile` 是 v1 仅有的 3 个由协议解释的 hidden-file 名；其他 `.xxx` 文件平台不解释（也不拒绝） |
| `scripts/` / `references/` / `assets/` | 名字标准化但**不强制必填** —— 作者按需创建。即使不存在加载器也不报错。 |

### 2.2 语义化命名指引（非强制，建议遵循）

| 目录 | 语义 | 何时用 |
| --- | --- | --- |
| `scripts/` | 沙箱内可执行的代码（Python `.py` / Node `.ts` / `.js` / shell 等） | 沙箱化 runtime（Tier 1 `<engine>::<lang>` / Tier 2 BYO）下，SKILL.md body 调用 `scripts/foo.py` 等 |
| `references/` | LLM 在执行过程中**渐进式**按需读取的参考文档（Markdown / 纯文本） | LLM 在 SKILL.md body 中指示「需要细节时读 `references/checklist.md`」；与 Claude Code "progressive disclosure" 概念一致 |
| `assets/` | 静态资源（图片、字体、文档模板、二进制等） | 脚本生成报告时引用 `assets/template.docx` 之类 |
| 其他 `<name>/` | 作者自定义 | 任意业务划分（如 `tests/` / `i18n/` 等），平台不解释 |

### 2.3 与 Claude Code / Agent Skills 开放标准的差异

* 开放标准（agentskills.io）只规定「use relative paths from the skill root」，未约束目录名 —— 本文档**与之兼容**（标准目录是约定，非强制）；
* Claude Code 文档示例同样使用 `scripts/` / `references/`，命名与本文档对齐；
* 不引入 Claude Code 的 `tools/` 目录约定 —— TFRobotServer 工具白名单走 frontmatter `allowed-tools`（A1 §2.2 / §4.1），与目录无关；
* 不引入 `_meta.yaml` / `manifest.json` 等附加元信息文件 —— 元数据全部在 frontmatter（A1）或 `.skillenv`（A2）中表达。

## 3. v1 占位符全集

> 3 个占位符，全部以 `TFROBOT_` 前缀。每个在 SKILL.md body 中可写为 `$TFROBOT_<NAME>` 或 `${TFROBOT_<NAME>}`（任意作者偏好），同名以环境变量形式注入沙箱进程。

### 3.1 `$TFROBOT_SKILL_DIR` — SKILL 包根资源前缀

**v1 唯一跨 runtime 展开值不同的占位符**。

| Runtime 形态 | 展开值（示例） | 用法 |
| --- | --- | --- |
| Tier 1 / Tier 2 沙箱化 runtime（`cubesandbox::*` / `e2b::*` / `cubesandbox` / `e2b`） | **本地挂载路径**，如 `/skill` —— 由 C6（[TFRS-199](https://turingfocus.atlassian.net/browse/TFRS-199)）从 MinIO 挂载 | 脚本中 `open(f"{os.environ['TFROBOT_SKILL_DIR']}/references/foo.md")`；SKILL.md body 中 `$TFROBOT_SKILL_DIR/scripts/main.py` |
| Tier 0 `prompt-only` | **平台内部不透明 URI**，如 `tfs-skill://<tenant>/<skill>/<version>/` —— 由平台 `skills` 工具（§6.5）识别；亦可在有 S3 凭据的外部环境中下载使用 | LLM 通过 `skills(skill_name, path)` 工具读取（详见 §6.5）；也可在 SKILL.md body 中拼接到 URI 供 LLM 参考构造工具调用：`Read("$TFROBOT_SKILL_DIR/references/checklist.md")` |

**URI 格式属平台内部协议**（具体由 B 模块实施时决定，可能是 `tfs-skill://` / `minio://` / 不透明 token），A4 仅承诺：

* 它是一个**前缀**，可与包内相对路径（`references/foo.md`）拼接形成完整资源标识；
* 平台提供的 `skills` 工具（§6.5）与沙箱本地 FS 都能解析；
* **作者不应解析此值的内部结构**，仅作为黑盒前缀使用；
* URI 可见性不构成安全漏洞 —— 任何资源访问最终都经 `skills` 工具或沙箱挂载层校验"当前 SKILL 范围 + 路径合法 + 黑名单"（§1.1 第 11 条 + §6.5）。

**与相对路径写法的关系**：开放标准推荐相对路径（`references/foo.md`）；平台 `skills` 工具同时支持「相对路径 + 当前激活 skill 上下文」与「完整 URI」两种入参，由作者按场景偏好选：

```
方式 A（推荐，开放标准对齐）：    skills(<implicit current skill>, "references/checklist.md")
方式 B（显式跨 SKILL 引用）：      skills("common-utils", "references/glossary.md")
方式 C（URI 拼接 + 外部环境）：    curl --signature $TFROBOT_SKILL_DIR/references/checklist.md  # 如作者接入有 S3 凭据的 Bash 环境
```

A4 不规定作者偏好；平台三种都支持。

### 3.2 `$TFROBOT_SESSION_ID` — 当前会话 ID

| 维度 | 契约 |
| --- | --- |
| 展开值 | 字符串。当前 robot 会话的唯一标识（具体格式由 robot 平台决定，通常 UUID v4） |
| 作用域 | 单次 SKILL 调用绑定到一个会话；该值在整个调用生命周期内不变 |
| Runtime | 所有 runtime 均可用（含 `prompt-only`） |
| 典型用途 | 日志关联、跨服务追踪、回调时携带、user-facing message 关联 |

### 3.3 `$TFROBOT_ROBOT_ID` — Robot 实例 ID

| 维度 | 契约 |
| --- | --- |
| 展开值 | 字符串。当前激活该 SKILL 的 robot 实例标识 |
| 作用域 | 同 SESSION_ID；同一 robot 多个会话共享相同 ROBOT_ID |
| Runtime | 所有 runtime 均可用 |
| 典型用途 | 多租户审计、按 robot 路由、工单关联 |

### 3.4 不收入 v1 占位符集合的候选

| 候选 | 不收入理由 |
| --- | --- |
| `$ARGUMENTS` / `$N` / `$name` | CLI 残留，由 A1 §3.1 整组排除；与 TFRobotServer 通过 LLM 工具调用结构化传参的入口本质不匹配 |
| `$TFROBOT_API_ENDPOINT` / `$TFROBOT_AUTH_TOKEN` 等 host 回调相关占位符 | **v1 采用纯函数沙箱脚本模型，无 host 回调通道**（§1 第 9 条）。脚本不通过 HTTP 调 TFRobotServer 主进程；需要"读会话 / 调其他工具 / 做对话决策"的场景应写为 `prompt-only` SKILL。回调模型推迟到出现具体业务诉求（详见 §9.3） |
| 用户 vault 解析后的明文（来自 `.skillenv`） | 敏感值隔离 —— 作为**沙箱进程 env** 注入（A2 §5 三层防护），不进入 SKILL.md body 展开路径 |
| `$TFROBOT_TENANT_ID` / `$TFROBOT_USER_ID` | 多租户/用户上下文 v1 暂不暴露给 SKILL 作者 —— 跨租户审计走平台日志而非 SKILL 自身；若 SKILL 真有按 tenant 区分行为的需求，应通过 robot 配置层显式传参，而非由 SKILL 解 tenant id 做分支 |
| `$TFROBOT_ARG` / `$TFROBOT_INPUT` 类输入占位符 | **输入参数注入语义**涉及 LLM 工具调用协议（结构化参数 vs 单字符串 vs 多入参等）；v1 纯函数模型下输入走 env vars / argv / stdin，具体格式由 C2（[TFRS-195](https://turingfocus.atlassian.net/browse/TFRS-195)）实施时定。A3 §3 例值中曾出现的 `$TFROBOT_ARG` 由 §9.2 反溯修订改写为通用表述 |
| `$TFROBOT_EFFORT` / `$TFROBOT_MODEL` | Claude Code 的 `${CLAUDE_EFFORT}` / model 注入是 CLI 选项注入；TFRobotServer 模型与算力档由平台按租户/计费策略决定（A1 §3.1），不暴露给 SKILL 作者 |
| `$TFROBOT_REQUEST_ID` | 可观测性需求由平台日志/trace 携带，无需 SKILL 自渲染；待具体调试场景出现再评估 |

## 4. 展开规则

### 4.1 SKILL.md body 中的展开

| 项 | 规则 |
| --- | --- |
| **时机** | 平台在把 SKILL.md body 拼到 LLM prompt **之前**完成替换；LLM 见到的是已展开的字符串 |
| **语法** | 接受 `$TFROBOT_NAME` 与 `${TFROBOT_NAME}` 两种 shell 风格；二者等价 |
| **命名空间** | 仅识别以 `$TFROBOT_` 开头的占位符；其他 `$VAR`（如 `$HOME` / `$PATH` / `$ARGUMENTS`）**保持字面原样**，平台不展开、不报错 |
| **未定义占位符** | 引用 v1 集合外的 `$TFROBOT_*`（如 `$TFROBOT_FOOBAR`）→ 加载器报错（"unknown placeholder $TFROBOT_FOOBAR"），避免作者误以为有该值 |
| **runtime 不匹配占位符** | 如 `prompt-only` SKILL.md body 中出现 `$TFROBOT_API_ENDPOINT` → 加载器报错（§3.4） |
| **转义** | 不支持。若作者真的想在 body 中写字面字符串 `$TFROBOT_SKILL_DIR`（极罕见），用 markdown code block 或文本说明绕开，本文档不引入转义语法（v1 克制） |
| **多次出现** | 同一占位符可出现任意次；每次独立展开为相同值 |

### 4.2 沙箱进程环境变量

| 项 | 规则 |
| --- | --- |
| **注入名** | 占位符名去掉 `$` / `${...}`，直接作为 env var 名（如 `TFROBOT_SKILL_DIR`、`TFROBOT_SESSION_ID`） |
| **注入时机** | 由 C2（[TFRS-195](https://turingfocus.atlassian.net/browse/TFRS-195)）在 `sandbox.create(envs=...)` 时一次性注入 |
| **与 `.skillenv` 注入的关系** | `.skillenv` 解析结果（A2 §4）+ `TFROBOT_*` 占位符 = 沙箱进程完整 env 集合；同名冲突时（虽然极罕见 —— `TFROBOT_` 前缀命名空间避让），`.skillenv` 优先级低（不允许覆盖平台占位符） |
| **`prompt-only` 下** | 无沙箱进程，env 注入路径 N/A；占位符仅在 SKILL.md body 中展开 |
| **作者读取** | Python：`os.environ["TFROBOT_SKILL_DIR"]`；Node：`process.env.TFROBOT_SKILL_DIR`；shell：`$TFROBOT_SKILL_DIR` |

### 4.3 展开示例

SKILL.md body 原文：

```markdown
## 任务
1. 读取 references/checklist.md（会话 ID：$TFROBOT_SESSION_ID）
2. 调用 scripts/process.py，输出落到 $TFROBOT_SKILL_DIR/out/
3. 把脚本 stdout 中的处理统计汇总到响应
```

`runtime: cubesandbox::python` + 平台展开后 LLM 看到：

```markdown
## 任务
1. 读取 references/checklist.md（会话 ID：8b0e4d2a-...）
2. 调用 scripts/process.py，输出落到 /skill/out/
3. 把脚本 stdout 中的处理统计汇总到响应
```

`runtime: prompt-only` + 平台展开后 LLM 看到：

```markdown
## 任务
1. 读取 references/checklist.md（会话 ID：8b0e4d2a-...）
2. 调用 scripts/process.py，输出落到 tfs-skill://tenant-a/my-skill/v1.2/out/
3. 把脚本 stdout 中的处理统计汇总到响应
```

> §3.1 示例：prompt-only 中 LLM 可把 `$TFROBOT_SKILL_DIR/scripts/process.py` 作为 Read 工具入参；具体能否执行非沙箱化代码取决于平台是否提供执行能力 —— v1 prompt-only 无此能力，作者应避免在 prompt-only SKILL 中引用 `scripts/`。

## 5. 与 Claude Code 占位符的兼容性

### 5.1 命名空间隔离

| 命名空间 | 拥有者 | 本文档处理 |
| --- | --- | --- |
| `$TFROBOT_*` | TFRobotServer（本文档） | 平台展开 |
| `${CLAUDE_SKILL_DIR}` / `${CLAUDE_SESSION_ID}` / `${CLAUDE_EFFORT}` 等 `CLAUDE_*` | Claude Code 私有扩展 | **保持字面**，平台不展开；跨客户端加载到 TFRobotServer 时 LLM 见到原文，自行理解（多数模型能从上下文推断含义） |
| `$ARGUMENTS` / `$1` / `$2` / `$name` | Claude Code 私有（CLI 残留） | **保持字面**，平台不展开（避免误展为空字符串造成 LLM 困惑） |

### 5.2 跨客户端互操作

* 在 TFRobotServer 编写的 SKILL.md 加载到 Claude Code 时：`$TFROBOT_*` 保持字面（Claude Code 不识别），LLM 见原文 —— 不影响 LLM 理解 body 大意；但若 SKILL 依赖具体值（如脚本路径），需要作者改写或仅在 TFRobotServer 内使用；
* 在 Claude Code 编写的 SKILL.md 加载到 TFRobotServer 时：`${CLAUDE_*}` / `$ARGUMENTS` 保持字面，LLM 见原文 —— 类似不影响理解，但依赖具体值的部分需作者重写为 `$TFROBOT_*`。

### 5.3 命名"为何不 mirror Claude Code 而用自有前缀"

* **物理上**：平台执行/挂载/会话语义与 Claude Code 不同（Agent-Computer 分离、MinIO 存储、多 Pod 部署），含义不能直接对齐；
* **逻辑上**：用 `CLAUDE_*` 借名容易让作者误以为语义完全一致；自有 `TFROBOT_*` 前缀**明确表达"这是 TFRobotServer 的占位符约定"**，配合本文档可一查便知；
* **互操作不破坏**：跨平台加载时各自保持字面，不引入冲突。

## 6. 与其他 A 子任务的同步

### 6.1 与 A1（frontmatter）对齐

* **A1 §4.2 留给 A4 的决策点**：「是否引入任何环境/运行时占位符、引入哪些、命名规则」 → 本文档定稿为 v1 4 个 `TFROBOT_*` 占位符 + 私有命名空间。A1 §4.2 中关于占位符的留白由本文档关闭。
* **A1 §3.1 已排除的 CLI 残留**：`$ARGUMENTS` / `$N` / `$name` 与本文档 §3.5 / §5.1 一致 —— 不收入、保持字面、跨客户端互操作时透传。

### 6.2 与 A2（`.skillenv`）对齐

* **`.skillenv` 文件名**：与本文档 §2 的「3 个 v1 特殊文件名」之一对齐；命名空间合并，便于加载器统一识别。
* **解析后注入的 env vars**：与 `TFROBOT_*` 占位符**共享沙箱进程 env 命名空间**，但**互不重叠**（用户 vault 命名空间通常是 `SLACK_BOT_TOKEN` / `NOTION_TOKEN` 等业务名，与平台 `TFROBOT_` 前缀避让）。
* **LLM 不可见性**：A2 §5 三层防护中"加载器黑名单"对 `.skillenv` 生效；A4 §3.5 把同样的"敏感值不暴露 body"原则推广到 `TFROBOT_AUTH_TOKEN` 等占位符。

### 6.3 与 A3（runtime 枚举）对齐

* **`.cubesandbox.dockerfile` / `.e2b.dockerfile` 文件名**：与本文档 §2 的「3 个 v1 特殊文件名」对齐；A3 §6.4 已定文件契约，本文档仅在目录结构中标注其位置。
* **跨 runtime 占位符可用性**：A3 §6.3 承诺占位符"在所有沙箱化 runtime 下统一可用" —— 本文档 §3 落地为 3 个具体占位符（`$TFROBOT_SKILL_DIR` / `$TFROBOT_SESSION_ID` / `$TFROBOT_ROBOT_ID`），全部跨 runtime 通用。
* **A3 例值中的 `$TFROBOT_ARG`**：A3 §3.2 / §3.3 例值中曾出现 `$TFROBOT_ARG`；本文档 §3.4 已说明此名属 C2 范围 —— A3 例值已在 §9.2 反溯就地修订。

### 6.4 沙箱脚本的纯函数输入/输出协议

v1 沙箱脚本是**无 host 回调**的纯函数（§1 第 9 条）；输入与输出协议如下：

| 通道 | 内容 | 实施 ticket |
| --- | --- | --- |
| **输入 · 环境变量** | `TFROBOT_SKILL_DIR` / `TFROBOT_SESSION_ID` / `TFROBOT_ROBOT_ID`（本文档 §3）+ `.skillenv` 解析后的明文（A2 §4） | C2 [TFRS-195](https://turingfocus.atlassian.net/browse/TFRS-195) |
| **输入 · 挂载目录** | SKILL 包目录挂载至 `$TFROBOT_SKILL_DIR`（沙箱本地 FS）；**MinIO → sandbox 卷的 staging 不经 FastAPI 主进程 FS**（§1.1 第 13 条），具体方案（baked image / lazy FUSE / sandbox 直拉 / pre-warm）由 C2/C6 ADR；staging 延迟约束："不淹没 < 60ms 冷启动"（CubeSandbox 60ms / E2B P95 ~90ms） | C6 [TFRS-199](https://turingfocus.atlassian.net/browse/TFRS-199) |
| **输入 · 启动参数** | LLM tool call 的结构化参数 → 注入方式（argv / stdin JSON / 额外 env）由 C2 定 | C2 [TFRS-195](https://turingfocus.atlassian.net/browse/TFRS-195) |
| **输出 · stdout** | 沙箱进程 stdout 捕获后作为 LLM 见到的 tool result | C2 / C6（执行链路） |
| **输出 · stderr** | 落平台日志，作者可查；不进入 LLM 上下文（除非 stdout 也写入） | C2 / C6 |
| **输出 · 产物文件** | 写入 `$TFROBOT_SKILL_DIR/out/` 的文件可选回收（具体策略由 C6 定） | C6 [TFRS-199](https://turingfocus.atlassian.net/browse/TFRS-199) |
| **❌ 无 host 回调** | 脚本不主动 HTTP 调 TFRobotServer 主进程；无 `$TFROBOT_API_ENDPOINT` | —（v1 不实现，[TFRS-197](https://turingfocus.atlassian.net/browse/TFRS-197) 暂停） |

需要"读会话 / 调其他工具 / 做对话决策"的 SKILL → 写为 `prompt-only`，由 LLM 在其工具循环中编排；不在沙箱脚本内做。

### 6.5 平台内置 `skills` 工具（LLM 读 SKILL 资源的唯一受控通道）

任何 SKILL 激活期间，LLM 隐式可用一个**平台内置工具** `skills`，用于读取 SKILL 包内文件（references/ / assets/ / scripts/ 等）。该工具是 prompt-only SKILL 访问自身资源、以及任何 SKILL 跨 skill 引用共享资源的**唯一受控通道** —— FastAPI 主进程不会绕过该工具替 LLM "直接 dump SKILL 内容"，符合 §1.1 第 11 条「SKILL 资源访问受控」。

| 项 | 设计 |
| --- | --- |
| **工具名** | `skills`（与 Claude Code 命名一致；跨平台肌肉记忆） |
| **入参 1** | `skill_name`: string —— 平台已注册 SKILL 的 `name`（A1 §2.1） |
| **入参 2** | `path`: string —— 相对 skill 根的路径（如 `references/foo.md` / `scripts/main.py` / `assets/template.docx`） |
| **返回** | 文件内容（文本直返；二进制走 base64） |
| **黑名单（强制 403）** | `.skillenv` / `.cubesandbox.dockerfile` / `.e2b.dockerfile` / 任何 `.*.dockerfile` 模式 / 任何 `.skillenv*` 模式 —— 与 §2.1 隐藏文件 + A2 §5 LLM 不可见性三层防护一致 |
| **路径校验** | 拒绝 `..` / 绝对路径 / 跳出 skill 根 / 符号链接逃逸 |
| **授权校验** | 当前会话有权访问 `skill_name` 才允许；权限模型由 B 模块 registry 定（[TFRS-181](https://turingfocus.atlassian.net/browse/TFRS-181)） |
| **大小上限** | 单次读取上限（建议 ~10 MB，超过返回错误并提示走 sandbox 处理）；具体阈值由 B 模块定 |
| **作用域** | **仅 LLM 工具调用层** —— 不暴露给 sandbox 脚本（脚本走本地挂载，§6.4） |
| **激活机制** | 平台内置工具，**任何** SKILL 激活时**隐式可用**；不要求作者写进 `allowed-tools`、也不能通过 `allowed-tools` 禁用（与 SKILL 激活机制本身同级）。底层依赖 TFRobotV2 Drive 层「隐身工具 + 动态暴露」能力（[TFROB-477](https://turingfocus.atlassian.net/browse/TFROB-477)）|
| **跨 SKILL 读** | 天然支持 —— `skill_name` 是显式入参，授权校验通过即可读其他 skill 资源。这覆盖"shared template / 共享术语表 / SKILL 间引用"场景 |

**典型调用**：

```python
# LLM 在 prompt-only SKILL 中按 SKILL.md body 指示读取 references/checklist.md
skills(skill_name="deploy-helper", path="references/checklist.md")

# 跨 SKILL 引用共享资源
skills(skill_name="common-utils", path="references/glossary.md")

# 黑名单触发拒绝
skills(skill_name="csv-aggregator", path=".skillenv")
# → 403 access denied: protected file
```

**实施 ticket**：[TFRS-181](https://turingfocus.atlassian.net/browse/TFRS-181)（B 模块）—— 见 §9.6。

## 7. 完整示例

### 7.1 极简 `prompt-only`（无脚本、无环境变量、纯参考文档驱动）

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

> 步骤 2 LLM 调用 `skills("deploy-helper", "references/checklist.md")`（§6.5）取回内容；`$TFROBOT_SESSION_ID` 在激活时展开为具体 UUID。

### 7.2 `cubesandbox::python` 沙箱（脚本 + 资源 + 纯函数输出）

包目录：

```
csv-aggregator/
├── SKILL.md
├── .skillenv
├── scripts/
│   ├── main.py
│   └── lib/normalize.py
├── references/
│   └── column-mapping.md
└── assets/
    └── report-template.docx
```

`SKILL.md`：

```yaml
---
name: csv-aggregator
description: 把多个 CSV 文件按用户给定的列规则聚合，并按 assets/report-template.docx 模板生成报告。
runtime: cubesandbox::python
compatibility: 已预装 pandas / python-docx；列规则参见 references/column-mapping.md
---

## 执行

1. 沙箱启动后运行 `$TFROBOT_SKILL_DIR/scripts/main.py`
2. 脚本把处理统计 JSON 写到 stdout；LLM 收到 stdout 作为 tool result，决定下一步
```

`.skillenv`：

```dotenv
LOG_LEVEL=INFO
```

`scripts/main.py`（沙箱内执行 —— 纯函数：env / 挂载 in，stdout / 文件 out）：

```python
import os, json, sys

skill_dir = os.environ["TFROBOT_SKILL_DIR"]
session = os.environ["TFROBOT_SESSION_ID"]

# 读包内资源
with open(f"{skill_dir}/assets/report-template.docx", "rb") as f:
    template = f.read()

# ... 业务逻辑 ...

# 输出统计到 stdout，被 LLM 看到
json.dump({"session": session, "rows_processed": 1234, "output_file": f"{skill_dir}/out/report.docx"}, sys.stdout)
```

### 7.3 `cubesandbox` BYO（含 ffmpeg）

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
description: 对上传的视频抽取关键帧并生成缩略图。
runtime: cubesandbox
compatibility: 自带镜像含 ffmpeg 7.x（详见 .cubesandbox.dockerfile）
---

运行 `$TFROBOT_SKILL_DIR/scripts/thumb.py`；脚本将缩略图 base64 + 元数据 JSON 写到 stdout，LLM 收到后输出给用户。
```

`.cubesandbox.dockerfile`（A3 §7.5 示例略）。

## 8. 准出对照

完成标准（来自 [TFRS-186](https://turingfocus.atlassian.net/browse/TFRS-186) 任务描述）：

* [x] 标准目录结构图（§2）
* [x] 占位符全集与展开规则（§3 + §4）
* [x] 与 Claude Code 占位符的兼容性说明（§5）

**额外澄清**（任务描述未列但本设计必须显式表达）：

* [x] 占位符私有命名空间策略（`TFROBOT_*` 前缀，§1.3 / §5）
* [x] `$TFROBOT_SKILL_DIR` 跨 runtime 同名异值的统一语义（§3.1）
* [x] **SKILL 资源访问安全模型**（§1.1 三条：FastAPI 不持久化 SKILL + 资源访问受控 + Sandbox staging 不经 FastAPI FS）
* [x] **平台内置 `skills` 工具契约**（§6.5）—— LLM 读 SKILL 资源的唯一受控通道；2 参数（`skill_name`, `path`）+ 黑名单（`.skillenv` / `.*.dockerfile`）+ 路径校验 + 授权
* [x] **纯函数沙箱脚本模型**（§1 第 9 条 / §6.4）—— v1 无 host 回调，与 E2B / CubeSandbox 行业默认一致
* [x] 敏感值不进入占位符集合的隔离原则（§1.8 / §3.4）
* [x] 与已定稿前序子任务的命名空间合并（`.skillenv` / `.cubesandbox.dockerfile` / `.e2b.dockerfile`，§2.1 / §6）

依赖下游子任务：

* `$TFROBOT_SKILL_DIR` 在沙箱中的具体挂载机制、占位符 env 注入时机 → [TFRS-199](https://turingfocus.atlassian.net/browse/TFRS-199)（C6）
* 沙箱脚本输入注入（env / argv / stdin）与输出捕获（stdout / 文件）实施 → [TFRS-195](https://turingfocus.atlassian.net/browse/TFRS-195)（C2）/ [TFRS-199](https://turingfocus.atlassian.net/browse/TFRS-199)（C6）
* **平台内置 `skills` 工具实施 + SKILL 资源访问安全模型 + MinIO → sandbox staging 管线** → 模块 B [TFRS-181](https://turingfocus.atlassian.net/browse/TFRS-181)（§9.6 新增 AC）
* **Staging 延迟约束（不淹没 < 60ms 冷启动）** → [TFRS-195](https://turingfocus.atlassian.net/browse/TFRS-195) C2 + [TFRS-199](https://turingfocus.atlassian.net/browse/TFRS-199) C6 联合 ADR（§9.5 / §9.7）

## 9. 任务描述 AC 修订建议（反溯）

A4 设计稿过程中发现以下上游 / 同级文档表述与本文档校准后的设计不一致，建议视为已澄清/废弃，由 Story owner（[TFRS-180](https://turingfocus.atlassian.net/browse/TFRS-180)）决定是否直接修订字段：

### 9.1 [TFRS-186](https://turingfocus.atlassian.net/browse/TFRS-186)（本任务）描述清理

| 原表述 | 建议修订 |
| --- | --- |
| 任务描述中列举的候选占位符 "`$TFROBOT_SKILL_DIR`、**`$ARGUMENTS`**、**`$N`**、`${TFROBOT_SESSION_ID}` 等" | 删除 `$ARGUMENTS` / `$N` —— 已由 A1 §3.1 整组排除（CLI 残留）；候选占位符列表改为 "`$TFROBOT_SKILL_DIR` / `$TFROBOT_SESSION_ID` / `$TFROBOT_ROBOT_ID`"（A4 §3 定稿 —— 3 个占位符，纯函数模型不含 `$TFROBOT_API_ENDPOINT`） |
| 完成标准「与 Claude Code 占位符的兼容性说明」 | 维持（A4 §5 已交付）；同时澄清"兼容"指**命名空间不冲突 + 跨客户端加载时各自保持字面**，**不**指 mirror Claude Code 命名 |

### 9.2 [A3 (TFRS-185)](https://turingfocus.atlassian.net/browse/TFRS-185) 文档同步小修（已就地修订）

| 位置 | 原表述 | 建议修订（**已在本 PR 应用**）|
| --- | --- | --- |
| A3 §3.2 「入参 / 出参」行 | "入参由 A4 占位符（如 `$TFROBOT_ARG`）注入" | "入参注入命名与机制由 C2 ([TFRS-195](https://turingfocus.atlassian.net/browse/TFRS-195)) 定（v1 纯函数模型，详见 A4 §3.4 / §6.4）；A4 仅承诺运行时上下文占位符" |
| A3 §7.2 / §7.4 例值中 `$TFROBOT_ARG` 出现处 | "从 `$TFROBOT_ARG` 读取页面 ID" / "从 `$TFROBOT_ARG` 读 thread URL" | 改写为「由 LLM 通过 C2 ([TFRS-195](https://turingfocus.atlassian.net/browse/TFRS-195)) 定的输入注入机制传入」 |

→ A3 文档已合入 develop（[#37](https://cnb.cool/turingfocus/tfrobotv2/TFRobotServer/-/pulls/37)）。本次 A4 PR 一并提交上述 A3 例值修订。

### 9.3 [TFRS-197](https://turingfocus.atlassian.net/browse/TFRS-197)（C4 鉴权通道）暂停

| 当前 ticket 范围 | 建议处理 |
| --- | --- |
| "SKILL 在 E2B 内调 TFRobotServer 内部 API（如读会话、调内置工具）的鉴权机制；鉴权 token 格式、Token 生命周期、工具白名单执行点" | **v1 暂停推进**。A4 §1 第 9 条 / §6.4 定稿沙箱脚本采用**纯函数模型**（无 host 回调），脚本不调 TFRobotServer 内部 API；本 ticket 整体推到 v2，等"脚本期间需要调其他工具 / 读会话"的具体业务诉求出现再重启。建议状态改为 `Won't do (v1) / Deferred to v2`，标签加 `deferred-v2`。Story owner 确认后归档。 |

### 9.4 [TFRS-182](https://turingfocus.atlassian.net/browse/TFRS-182)（Story 模块 C）AC 项删除

| 原 AC | 建议处理 |
| --- | --- |
| "Skill 内回调 TFRobotServer 内部 API 的鉴权通道（短期 token、绑会话与租户）（C4 [TFRS-197](https://turingfocus.atlassian.net/browse/TFRS-197)）" | **删除**该 AC 项 —— v1 纯函数模型下不存在此通道；C4 ticket 暂停。 |
| "内部 RPC 回调通道" 在「背景与动机」中的措辞 | 删除或改写为 "v1 采用纯函数沙箱脚本模型，无 host 回调"。 |

### 9.5 [TFRS-195](https://turingfocus.atlassian.net/browse/TFRS-195)（C2 注入链路）AC 扩项

| 原 AC | 建议增加 |
| --- | --- |
| "注入流程图（读 .skillenv → 查密钥库 → 注入容器）" | **增加**："① 沙箱脚本启动参数 / stdin / argv 注入机制的最终选型 —— 承接 A4 §6.4 纯函数模型 I/O 协议中"启动参数"行的延后决定；② staging 延迟约束：MinIO → sandbox 卷的传递不能淹没 < 60ms 冷启动（CubeSandbox 60ms / E2B P95 ~90ms），具体实现方向（per-skill-version baked image / lazy FUSE mount / sandbox 直拉 + signed URL / speculative pre-warm）由 C2/C6 联合出 ADR" |

### 9.6 [TFRS-181](https://turingfocus.atlassian.net/browse/TFRS-181)（B 模块存储）AC 扩项

A4 在 §1.1 / §6.5 引入了两个 B 模块负责的新能力，本节统一登记：

| 新增 AC | 描述 |
| --- | --- |
| **平台内置 `skills` 工具实施** | LLM 端读 SKILL 资源的唯一受控通道（A4 §6.5）。范围：① 入参 `(skill_name, path)` 双参数签名；② 黑名单（`.skillenv` / `.*.dockerfile`）强制 403；③ 路径校验（拒 `..` / 绝对路径 / 跳出 skill 根 / 符号链接逃逸）；④ 授权校验（基于 registry 权限模型）；⑤ 大小上限（建议 ~10 MB）；⑥ 与 LLM 工具循环的注册机制（每次 SKILL 激活时隐式可用，作者不可禁用） |
| **SKILL 资源访问安全模型** | 落实 A4 §1.1 三条安全原则：① FastAPI 主进程不持久化 SKILL（只读字节、不下盘、不 eval）；② 跨 SKILL 访问的 registry 授权矩阵（决定哪些会话有权读哪些 SKILL）；③ 黑名单与路径校验逻辑下沉到加载器，避免每个工具重写 |
| **Sandbox staging 管线** | MinIO → sandbox 卷的文件传递管线设计（与 C6 协作）。**约束**：不经 FastAPI 主进程 FS（A4 §1.1 第 13 条）+ 延迟不淹没 < 60ms 冷启动（A4 §9.5）。实现方向同 §9.5（baked image / lazy FUSE / 直拉 / pre-warm 任选） |

### 9.7 [TFRS-199](https://turingfocus.atlassian.net/browse/TFRS-199)（C6 占位符 + 挂载）AC 扩项

| 原 AC | 建议增加 |
| --- | --- |
| "挂载机制（MinIO → 容器内 `/skill/` 路径）" | **增加**：与 B 模块（[TFRS-181](https://turingfocus.atlassian.net/browse/TFRS-181) §9.6）staging 管线协作，约束：① **不经 FastAPI 主进程 FS**（A4 §1.1 第 13 条）；② **延迟不淹没 < 60ms 冷启动**（A4 §9.5）；具体方案与 C2 联合出 ADR |

### 9.8 跨项目依赖：TFRobotV2 [TFROB-477](https://turingfocus.atlassian.net/browse/TFROB-477)（Drive 层隐身工具 + 动态暴露）

A4 §6.5 `skills` 工具的「平台内置 + SKILL 激活时隐式可用 + 不通过 `allowed-tools` 暴露」机制，本质需要 TFRobotV2 Drive 层支持「工具默认隐身 + 上下文动态暴露」能力 —— 该能力已作为 Feature Request 提交：

* **Issue**：[TFROB-477](https://turingfocus.atlassian.net/browse/TFROB-477)（TFRobotV2 项目，Story 类型，P1 重要）
* **请求范围**：visibility 属性、Query 接口默认过滤、运行时动态暴露、暴露 scope（建议会话级 sticky）、隐身 ≠ 免授权
* **本 A4 文档对其依赖**：
  - §6.5 `skills` 工具实施依赖 hidden tool 机制（不被 LLM 主动看见，但激活时可调用）
  - 未来 `SkillSandbox` 工具（Module C 执行入口）同样依赖此机制 —— `runtime != prompt-only` 的 SKILL 激活时动态暴露给 LLM
* **阻塞关系**：
  - **不阻塞** A4 本身定稿（仅协议层）
  - **不阻塞** SKILL 协议 v1 文档化（A4 / A5）
  - **阻塞** Module C 端到端实施时（[TFRS-182](https://turingfocus.atlassian.net/browse/TFRS-182) / [TFRS-194](https://turingfocus.atlassian.net/browse/TFRS-194)）：当 [TFRS-194](https://turingfocus.atlassian.net/browse/TFRS-194) C1 准备落地 `SkillSandbox` 工具时，必须等 [TFROB-477](https://turingfocus.atlassian.net/browse/TFROB-477) 在 TFRobotV2 中实现并发版
* **回退策略**（如 [TFROB-477](https://turingfocus.atlassian.net/browse/TFROB-477) 推迟）：本地 hack 一层 visibility wrapper（拦截 `list_tools` 返回值过滤 hidden 工具），临时支撑 v1，但语义边界不如内核原生支持清晰

**协议设计教训 —— 占位符命名空间策略**：

设计 SKILL 协议这类有跨客户端互操作可能的协议时，「是否 mirror 上游标准的占位符命名」是反复出现的张力。**完全 mirror**（用 `${CLAUDE_*}`）让作者跨平台零迁移成本，但平台行为不一致会埋"看似相同实则不同"的隐 bug；**完全自有**（用 `${TFROBOT_*}`）让跨平台需要重写部分内容，但语义边界清晰。本文档选择后者 + "未识别占位符保持字面"的容错策略，既清晰又不破坏互操作。后续 Module A 类任务可借鉴这种"私有命名空间 + 跨平台字面透传"模式。

**架构教训 —— 沙箱脚本的工具表面**：

A4 设计稿过程中识别出一个潜在过度设计：A3 / 早期 A4 / TFRS-197 都假设沙箱脚本需要 HTTP 回调主进程读会话、调工具，与 E2B / CubeSandbox 行业默认（脚本 = 纯函数）背离。回归审视后改回纯函数模型 —— **能力 (B) LLM 工具调用与 (C) 脚本执行严格分层**，需要跨层调度的场景由 LLM 用 prompt-only SKILL 自己编排，而非脚本回头调度。这避免了：① 鉴权复杂度（C4 整张 ticket 消失）；② 两个白名单的一致性维护；③ "沙箱化是为了隔离，但脚本又能回头调平台"的语义矛盾。后续 Module C / 跨进程协作设计可参考此分层原则。

**架构教训 —— SKILL 资源的"不可信资产"边界**：

A4 后期补足了 §1.1 三条安全原则与 §6.5 `skills` 工具契约 —— 之前 §3.1 把 `prompt-only` 下 `$TFROBOT_SKILL_DIR` 展开为 URI 时一度让设计稍显"敞开"。澄清后的边界：**SKILL 是不可信资产，但占位符 URI 可见并不构成漏洞** —— 真正的安全闸门在 `skills` 工具实现层（黑名单 + 路径校验 + 授权 + 大小上限）；URI 字面给 LLM 看反而是个特性（作者在外部 Bash + S3 凭据环境可直接 `curl`）。教训：**设计安全协议时区分"信息边界"与"能力边界"** —— 暴露标识符不等于授权操作；闸门设在能力层，标识符可以放心透传。
