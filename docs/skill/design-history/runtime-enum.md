# A3 — SKILL.md `runtime` 字段枚举值定稿

> Jira：[TFRS-185](https://turingfocus.atlassian.net/browse/TFRS-185)（Story [TFRS-180](https://turingfocus.atlassian.net/browse/TFRS-180) / Epic [TFRS-179](https://turingfocus.atlassian.net/browse/TFRS-179)）
> 范围：仅 `runtime` 字段在 v1 的枚举值集合与各值的运行环境契约；具体镜像构建、模板版本管理由 [TFRS-194](https://turingfocus.atlassian.net/browse/TFRS-194)（C1）实施。
> 结论一句话：**`runtime` v1 共 7 个枚举值，全部显式引擎绑定 —— `prompt-only`（无沙箱）+ 4 个 `<engine>::<lang>` 平台预制（`cubesandbox::python` / `cubesandbox::node` / `e2b::python` / `e2b::node`）+ 2 个 `<engine>` BYO（`cubesandbox` / `e2b`，需配 `.cubesandbox.dockerfile` / `.e2b.dockerfile`）。平台同时运行两套引擎集群，作者按需选择**。

## 0. 重要前置框架性调整：双引擎并存，runtime 取值显式带引擎前缀

任务描述（[TFRS-185](https://turingfocus.atlassian.net/browse/TFRS-185)）原表述「runtime 枚举值与 **E2B 模板**的对齐」把协议层与某一个具体执行引擎钉死。设计过程中以下两点要求出现，需在协议层做框架性调整：

1. **服务部署在腾讯云**，更倾向使用同生态的开源沙箱方案 [CubeSandbox](https://github.com/TencentCloud/CubeSandbox)（腾讯云 2026-04-21 Apache-2.0 开源；RustVMM/KVM；`< 60 ms` 冷启动；`< 5 MB` 实例开销；**100% E2B SDK drop-in**）；
2. **封装要内核可换、统一接口** —— 现在选 CubeSandbox 为主、E2B 为备，未来可能再增引擎；协议层若钉死单一引擎，每次切换都要 SKILL 作者改 frontmatter，违背克制原则。

**据此调整为「同时支持双引擎，作者自选」的模型**：

| 层 | 适用场景 | 取值 | 引擎归属 | 镜像来源 | 必备文件 |
| --- | --- | --- | --- | --- | --- |
| **Tier 0** | 纯 prompt 编排 | `prompt-only` | 无沙箱 | — | — |
| **Tier 1 平台预制运行时** | 常见场景（绝大多数 SKILL） | `<engine>::<lang>` 形式 4 个：`cubesandbox::python` / `cubesandbox::node` / `e2b::python` / `e2b::node` | **由取值决定** | 平台为每个引擎维护一份预制镜像 | 无 |
| **Tier 2 BYO 自定义镜像** | 极端场景（小众语言 / 特定二进制 / 大依赖） | `<engine>` 形式 2 个：`cubesandbox` / `e2b` | **由取值决定** | 作者在 skill 包根放 `.cubesandbox.dockerfile` 或 `.e2b.dockerfile`，平台代为构建、注册模板 | 与 runtime 同名的 dockerfile |

设计要点：

- **平台同时运行两套引擎集群** —— CubeSandbox（v1 主选，国内主集群）+ E2B（备选，覆盖出海 / 多 region / CubeSandbox 故障应急）。两套始终在线，作者按需选择。
- **「内核可换」由"作者改一个前缀"承担**：原先想法是"平台一项配置切换所有 SKILL"，但实际上不同 SKILL 对引擎的偏好不一样（部分要 CubeSandbox 极低延迟，部分要 E2B 已有的特定镜像），把选择权交还作者更诚实。
- **协议层与后端 1:1 对应** —— `cubesandbox::python` 在后端就是 CubeSandbox 集群里的一份具体 template_id；不需要中间映射表。后端管理大幅简化。
- **`<engine>::<lang>` 与 `<engine>` 的语法对称性** —— `::` 后是平台预制的 lang stack，去掉 `::lang` 退化为 BYO；同样 Dockerfile 文件名 `.cubesandbox.dockerfile` / `.e2b.dockerfile` 也带引擎前缀，与 runtime 取值一一对应。
- **CubeSandbox / E2B SDK drop-in 仍然成立** —— 内部统一走 `e2b-code-interpreter` SDK，仅 `E2B_API_URL` + `template_id` 路由差异；执行器代码一份。

## 1. 裁定原则

1. **协议层与引擎 1:1 对应**：所有沙箱化 runtime 取值都显式包含引擎名（`cubesandbox` / `e2b`），不存在引擎中立的"抽象语言"取值。`prompt-only` 例外（无沙箱）。
2. **同时支持两套引擎集群**：平台同时运行 CubeSandbox（v1 主选）+ E2B（备选），始终在线；作者按需选哪个。CubeSandbox 不可用时 E2B 自然作为应急；不存在「全平台切到 E2B」的运维一刀切。
3. **`::` 区分平台预制 vs BYO**：`<engine>::<lang>` = 平台预制 lang stack；`<engine>`（无 `::`）= 作者自带 Dockerfile。Dockerfile 文件名 `.<engine>.dockerfile` 与 `<engine>` 部分严格对应。
4. **强契约（runtime ↔ Dockerfile 文件名）**：
   - `runtime: cubesandbox` ⇔ skill 包根**必须**有 `.cubesandbox.dockerfile`、且**不允许**有 `.e2b.dockerfile`
   - `runtime: e2b` ⇔ 反之
   - `runtime: <engine>::<lang>` 任一取值 ⇔ **不允许**任何 `.<engine>.dockerfile`
   - `runtime: prompt-only` ⇔ 同上 + 也不允许 `.skillenv`
   - 任一条违反 → 加载器报错
5. **"内置依赖"字面承诺、不暗装第三方包**：每个 `<engine>::<lang>` 仅承诺**列在本文档中**的库存在；不在表内的库由 SKILL 作者通过 `subprocess`/`pip install` / `npm install` 在沙箱内自装（带后果）。需要确定性预装的作者走 Tier 2 BYO，把依赖固化进 Dockerfile。
6. **`python` 单一预制 lang（不再细分 `python-data`）**：与 CubeSandbox 官方 `sandbox-code` / E2B `code-interpreter` 行业默认一致 —— 一份 Python 沙箱包含通用胶水栈 + pandas/numpy 数据栈，覆盖 80%+ 场景；warm-template 抹平镜像体积差异。需要更激进资源（>2 vCPU / >2 GB）或专用栈（PyTorch / OCR）走 Tier 2 BYO。
7. **v1 仍克制**：Tier 1 收 4 个常见预制（2 引擎 × 2 lang），Tier 2 给出 2 个 BYO 出口；**不收**：`shell` / `mcp-server` / `python-ml` / `python-browser` / `python-data`（已并入 `python`）等 —— 这些场景由 Tier 2 BYO 表达即可。
8. **未来跨引擎可移植性的预留**：当前 SKILL 一次声明一个 runtime（绑一个引擎）；将来若有「同一 SKILL 想同时支持两引擎」的需求，预留扩展空间 —— `runtime` 可演进为列表（如 `runtime: [cubesandbox, e2b]`）配合两个引擎的 Dockerfile 都在包内、由平台按可用性挑一个。v1 不实现，但当前文件名/取值方案天然兼容。

## 2. v1 枚举值表

### 2.1 Tier 0 — 无沙箱（1 个）

| 值 | 是否沙箱 | 用途 | 是否默认 |
| --- | --- | --- | --- |
| `prompt-only` | 否 | 纯 prompt + LLM 工具调用编排；不需要执行任意脚本 | ✅ 默认（与 A1 §2.3 一致） |

### 2.2 Tier 1 — 平台预制运行时（4 个）

语法：`<engine>::<lang>`。`engine` 决定走哪个集群，`lang` 决定平台预制的语言栈。

| 值 | 引擎 | 语言 / 解释器 | 用途 |
| --- | --- | --- | --- |
| `cubesandbox::python` | CubeSandbox | Python 3.11+（含 pandas/numpy 数据栈） | CubeSandbox 上跑 Python 脚本（API 胶水 / 文件处理 / 数据分析 / 表格图表） |
| `cubesandbox::node` | CubeSandbox | Node.js 22+ | CubeSandbox 上跑 TS / JS |
| `e2b::python` | E2B | Python 3.11+（含 pandas/numpy 数据栈） | E2B 上跑 Python 脚本（同上） |
| `e2b::node` | E2B | Node.js 22+ | E2B 上跑 TS / JS |

> Tier 1 镜像由 C1（[TFRS-194](https://turingfocus.atlassian.net/browse/TFRS-194)）在 CubeSandbox 与 E2B 两端**分别**维护，共 4 份镜像。同一 `lang` 在两个引擎下的契约（§3）一致，作者可按集群偏好自由选 `cubesandbox::*` 或 `e2b::*`。`python` 单一预制包含通用胶水栈 + pandas/numpy 数据栈，与 CubeSandbox 官方 `sandbox-code` / E2B `code-interpreter` 行业默认一致。

### 2.3 Tier 2 — 引擎绑定 BYO（2 个）

| 值 | 目标引擎 | 必备文件 | 不允许文件 | 镜像规范 |
| --- | --- | --- | --- | --- |
| `cubesandbox` | CubeSandbox | skill 包根 `.cubesandbox.dockerfile` | `.e2b.dockerfile` | 必须嵌入 `envd:49983`（`FROM ghcr.io/tencentcloud/cubesandbox-base` 或 `COPY --from=` 注入），详见 [CubeSandbox BYO 文档](https://github.com/TencentCloud/CubeSandbox/blob/master/docs/guide/tutorials/bring-your-own-image.md) |
| `e2b` | E2B | skill 包根 `.e2b.dockerfile` | `.cubesandbox.dockerfile` | 必须符合 [E2B Template spec](https://e2b.dev/docs/sandbox-template)（`e2b.toml` 由平台代生成） |

> Tier 2 镜像每个 skill 版本一次性构建、由平台代为 build & push & 注册模板（具体管线由 C1 实施）；作者只负责 `.<engine>.dockerfile` 内容。

### 2.4 强契约：runtime 取值与 dockerfile 文件名的存在性绑定

| `runtime` 取值 | `.cubesandbox.dockerfile` | `.e2b.dockerfile` | 加载器行为 |
| --- | --- | --- | --- |
| `prompt-only` | 任一存在即拒绝 | 任一存在即拒绝 | 仅当两者都不存在时 ✅ 加载 |
| `<engine>::<lang>` 任一（Tier 1） | 任一存在即拒绝 | 任一存在即拒绝 | 仅当两者都不存在时 ✅ 加载 |
| `cubesandbox` | **必须存在** | 不允许 | ✅ 加载 |
| `e2b` | 不允许 | **必须存在** | ✅ 加载 |

> 加载器在 SKILL 解析阶段就强制此契约，避免"看似生效实际被忽略"的悬念。错误信息必须明确指出违反了哪条规则（例如「`runtime: cubesandbox` 要求存在 `.cubesandbox.dockerfile`，找到的却是 `.e2b.dockerfile`」）。

### 2.5 命名说明

- **冒号-冒号分隔符 `::`**：模仿 C++/Rust 命名空间符号，表达"该 lang 属于此 engine 的命名空间"；与文件路径 `/` 区分，避免与 OCI 镜像引用 / URI 等混淆。
- **不细分 `python-basic` / `python-data`**：单一 `python` 即包含数据栈，与行业默认（CubeSandbox `sandbox-code` / E2B `code-interpreter`）一致；细分场景由 Tier 2 BYO 承接。
- **去掉 `-sandbox` 后缀**：原 A1 §2.3 示例 `python-sandbox`/`node-sandbox` 已废弃；引擎前缀已足够表达"沙箱化"。

## 3. 各运行时契约

> 「契约」=作者可在该 runtime 下**直接 assume**的最小集合。沙箱具体镜像内容（PIP_INDEX / OS 包 / 写入层大小 / 探针端口）由 C1（[TFRS-194](https://turingfocus.atlassian.net/browse/TFRS-194)）定稿；本节只承诺**作者可见的运行时契约**。

### 3.1 `prompt-only`（默认）

| 维度 | 契约 |
| --- | --- |
| 沙箱 | **无**。SKILL.md 在 FastAPI 主进程内被加载、渲染并拼到 prompt；无任何脚本执行路径。 |
| 可用工具 | 由 frontmatter `allowed-tools`（A1 §2.2）+ robot 当前工具白名单交集决定；走 TFRobot drive / MCP / 内置工具的 LLM-工具循环。 |
| `.skillenv` | 仍可声明（A2 §2 可选）但**不注入到任何执行上下文**（无执行上下文）。v1 下：`prompt-only` SKILL 写 `.skillenv` 会被加载器拒绝并报错（避免作者误以为有沙箱）。 |
| 失败模式 | LLM 自然 fail 或 tool 调用错误；与已有 robot chat 路径一致。 |
| 启动开销 | 0（直接进 prompt 拼接）。 |

**对应 [TFRS-198](https://turingfocus.atlassian.net/browse/TFRS-198)（C5 prompt-only 快速路径）**：所有 `runtime: prompt-only` 的 SKILL 在 C5 路径下处理，不进任何 sandbox provisioner。

### 3.2 `cubesandbox::python` / `e2b::python`

> 同一 `lang` 在两个引擎下契约一致；下表通用。引擎差异（冷启动、隔离粒度、网络策略实现）参见 §4.1。

| 维度 | 契约 |
| --- | --- |
| 解释器 | Python 3.11 或更高（具体版本由 C1 定稿；保证 `>=3.11`） |
| stdlib | 完整 stdlib 可用 |
| 预装第三方 | 通用胶水栈：`requests` / `httpx` / `pydantic>=2` / `pyyaml` / `python-dateutil`<br/>数据栈：`pandas` / `numpy` / `matplotlib` / `openpyxl` / `python-docx` / `pillow`<br/>其他自装（`pip install ...` in-sandbox 或 BYO 走 Tier 2） |
| 文件系统 | 沙箱本地可读写；SKILL 包目录挂载到 `$TFROBOT_SKILL_DIR`（A4 [TFRS-186](https://turingfocus.atlassian.net/browse/TFRS-186) 定稿）；`/tmp` 至少 1 GB 可写空间 |
| 网络 | 默认允许出站 HTTPS（具体 egress 白名单由 C1 镜像策略 + CubeVS/eBPF（CubeSandbox）或模板 egress（E2B）决定）；入站不开放 |
| 入参 / 出参 | 入参注入命名与机制由 C4（[TFRS-197](https://turingfocus.atlassian.net/browse/TFRS-197)）定（详见 [A4 §3.5](directory-placeholders.md)）；A4 仅承诺 `$TFROBOT_SKILL_DIR` / `$TFROBOT_SESSION_ID` / `$TFROBOT_ROBOT_ID` / `$TFROBOT_API_ENDPOINT` 等运行时上下文占位符。出参约定走 stdout 或回调 API（C4 [TFRS-197](https://turingfocus.atlassian.net/browse/TFRS-197)） |
| 资源默认 | 1 vCPU / 1 GB / 8 min wall-clock —— 具体默认值与上限由 C1 定稿；本文档只承诺有缺省 |

**典型使用场景**：API 胶水（Notion / Slack / GitHub）、文件格式转换、Excel/CSV 解析与汇总、PDF 抽表、表格生成报告、简单图表 —— 覆盖通用 Python 场景。

**为什么不细分 `python-data` / `python-ml` / `python-pdf`**：

- 数据栈（pandas/numpy）已包含在 `python` 中（与 CubeSandbox `sandbox-code` / E2B `code-interpreter` 行业默认一致），无需独立运行时；
- 极重栈（PyTorch / OCR / Browser，GB 级镜像）冷启动 / 单机密度收益反比，且当前无具体 SKILL 需求；待出现明确场景由 Tier 2 BYO 承接，无需新增 Tier 1 枚举。

### 3.3 `cubesandbox::node` / `e2b::node`

| 维度 | 契约 |
| --- | --- |
| 解释器 | Node.js 22 LTS 或更高（具体版本由 C1 定稿；保证 `>=22`） |
| TypeScript 运行 | 预装全局 `tsx`（推荐入口）；可直接 `npx tsx scripts/index.ts` 跑 TS 源；如需 `tsc` 编译为 JS，作者自行装 `typescript` |
| 预装第三方 | 全局：`tsx` / `zod` / `axios` / `dayjs`；其他自装（`npm install ...` in-sandbox 或 BYO 走 Tier 2） |
| package.json | 可选 —— 作者若提供，沙箱启动时**不自动** `npm install`（避免冷启动膨胀）；如需启动期装包，作者在脚本里显式调 `npm ci` |
| 文件系统 | 沙箱本地可读写；SKILL 包目录挂载到 `$TFROBOT_SKILL_DIR`（A4 定稿） |
| 网络 | 同 `python`：出站 HTTPS 默认允许，入站不开放 |
| 入参 / 出参 | 入参由 A4 占位符注入；出参约定走 stdout 或回调 API（C4 [TFRS-197](https://turingfocus.atlassian.net/browse/TFRS-197)） |
| 资源默认 | 1 vCPU / 512 MB / 5 min wall-clock —— 具体由 C1 定稿 |

**典型使用场景**：TS 脚本调三方 SaaS API（Notion/Slack/GitHub SDK 多用 TS）、前端构建链中的小工具复用、JSON Schema 校验、用 `zod` 解析 LLM 输出。

### 3.4 `cubesandbox` / `e2b`（Tier 2 BYO 自定义镜像）

| 维度 | 契约 |
| --- | --- |
| 镜像来源 | 作者提供 skill 包根 `.cubesandbox.dockerfile`（对应 `runtime: cubesandbox`）或 `.e2b.dockerfile`（对应 `runtime: e2b`）；平台代为 build & push & 注册模板（管线由 C1 实施） |
| 镜像规范 | `cubesandbox` → 必须嵌 `envd:49983` 守护进程，参照官方 BYO 文档；`e2b` → 必须符合 E2B template spec |
| 可装内容 | **任意** —— 任何 apt 包、任何语言运行时、专用二进制；唯一约束是镜像规范 |
| 文件系统 | 与 Tier 1 一致：`$TFROBOT_SKILL_DIR` 挂载、可读写 |
| 网络 | 与 Tier 1 一致 |
| `.skillenv` 注入 | 与 Tier 1 一致 —— 在主进程解析后通过 SDK `envs=...` 注入，与 Dockerfile 内 `ENV` 指令叠加（同名冲突时 `envs=` 覆盖） |
| 入参 / 出参 | 与 Tier 1 一致 |
| 资源默认 | 与 Tier 1 同档；若 Dockerfile 标注更高资源需求由 C1 定上限 |
| 跨引擎可换 | **否**。`runtime: cubesandbox` 的 skill 只在 CubeSandbox 集群运行；`runtime: e2b` 只在 E2B 集群运行 —— 平台不做自动转译。同一 skill 若想跨两个引擎，需作者改 `runtime` 取值（并切换对应 Dockerfile）。 |
| 镜像构建失败 | skill 上传/发布阶段即报错；构建产物（artifact + template_id）入 Module B registry，与 skill 版本一一绑定 |

**典型使用场景**：

- 需要特定二进制（如 ffmpeg / chromium / 私有 SDK）的 SKILL；
- 大依赖（>200 MB）的数据/ML SKILL —— 比每次冷启动装包快得多；
- 跨语言（Go/Rust binary + Python wrapper）；
- 已有的、想直接复用的内部容器镜像。

**作者不应走 Tier 2 的场景**：只想在 `<engine>::python` 上加一两个 pip 包 —— 在 SKILL 脚本里 `pip install` 即可，无需 BYO（BYO 会失去 warm-template 红利、增加构建延迟）。

## 4. 与执行引擎的边界

> **核心约定**：SKILL 协议（本文档）只定义"运行时契约"；具体由哪个引擎落地、版本如何切换、模板 ID 如何映射，**全部**属模块 C 范围。本节只给出对照说明与切换机制，便于 C1 落地。

### 4.1 引擎对照（v1）

CubeSandbox 与 E2B 在 SDK 与能力面平价（均支持文件 I/O / 进程 / pause-resume / 浏览器 / 网络策略 / RL），差异仅在以下维度：

| 维度 | CubeSandbox（v1 主选） | E2B（备选） |
| --- | --- | --- |
| 协议 | E2B SDK drop-in（换 `E2B_API_URL` 即切换） | 原生 |
| 隔离 | RustVMM + KVM 独立 guest kernel | gVisor / Firecracker |
| 冷启动 | < 60 ms（P95 90 ms） | 100–300 ms 典型 |
| 单实例开销 | < 5 MB | 数百 MB |
| 网络隔离 | eBPF/CubeVS 细粒度租户隔离 | 模板预设 egress |
| 国内可用 | 与 TFRobotServer 同区，国内镜像 `cube-sandbox-cn.tencentcloudcr.com` | 镜像仓与代码仓均在境外，国内合规须自行评估 |
| 自托管难度 | 一键脚本；普通腾讯云 VM 通过 PVM 即可跑（无需 bare-metal / nested-virt） | [`e2b-dev/infra`](https://github.com/e2b-dev/infra) Terraform + Packer + Nomad；**官方仅支持 GCP（GA） / AWS（Beta）**，腾讯云需自行 fork module；最低 ~2500 GB SSD + 24 CPU 起步；需 Cloudflare 账号 |
| 部署成本 | 自建免费（KVM 主机 / PVM 云 VM） | 自建免费（OSS）/ SaaS 按时长计费 |
| 首发模板 | `sandbox-code`（Python interpreter），其他需自建 | 多个官方（`python-basic` / `python-data` / `node` 等） |
| 许可 | Apache-2.0 | Apache-2.0（[`e2b-dev/infra`](https://github.com/e2b-dev/infra)） |

→ v1 选 CubeSandbox 为主、E2B 为备的理由：

1. **部署到腾讯云的工程复杂度低** —— CubeSandbox 一键脚本 + PVM 兼容普通云 VM；E2B 官方 Terraform 仅覆盖 GCP / AWS，腾讯云需自行 fork module + 准备 Cloudflare + Nomad 集群。
2. **国内生态对齐** —— CubeSandbox 镜像源在国内（`cube-sandbox-cn.tencentcloudcr.com`），E2B 镜像 / 代码仓均在境外。
3. **资源底盘性能** —— 5 MB / 实例 vs E2B 数百 MB；冷启动 < 60 ms vs 100–300 ms；小集群即可起步，E2B 推荐起点 ~2500 GB SSD + 24 CPU。
4. **腾讯云原厂支持** —— 与 TFRobotServer 同生态，工单 / 合规 / 国内沟通顺畅。

E2B 在 v1 的定位：

- **常驻备选集群**：与 CubeSandbox 同时在线，作者可在 SKILL 中选 `e2b::*` / `e2b`，覆盖海外 region、合规要求、CubeSandbox 故障应急等场景；
- **作者自选粒度**：选哪个引擎是 per-SKILL 决定（runtime 取值显式表达），而非全局开关；这与「同时支持双引擎」的设计一致。

执行器代码因 CubeSandbox SDK drop-in 仍是一份（同 §4.3）；运维成本主要在两套集群的部署与监控，由 [TFRS-194](https://turingfocus.atlassian.net/browse/TFRS-194) C1 / OPS 落地评估。

### 4.2 引擎选择机制

```
                         ┌─ Tier 1（平台预制） ────────────────────────────────────────────────┐
                         │                                                                     │
runtime: cubesandbox::python  ──► CubeSandbox 集群 + 预制 python 模板 (cube-tpl-py)             │
runtime: cubesandbox::node    ──► CubeSandbox 集群 + 预制 node 模板    (cube-tpl-node)          │
runtime: e2b::python          ──► E2B 集群        + 预制 python 模板  (e2b-python)              │
runtime: e2b::node            ──► E2B 集群        + 预制 node 模板    (e2b-node)                │
                         │                                                                     │
                         │   静态映射 { runtime_value → (engine, template_id) }                │
                         │   C1 维护：CubeSandbox / E2B 两端各 2 份镜像，共 4 份               │
                         └─────────────────────────────────────────────────────────────────────┘

                         ┌─ Tier 2（BYO） ────────────────────────────────────────────────────┐
runtime: cubesandbox     │                                                                     │
+ .cubesandbox.dockerfile│──► CubeSandbox 集群 + 平台代构建的镜像（每个 skill 版本一次）        │
                         │                                                                     │
runtime: e2b             │                                                                     │
+ .e2b.dockerfile        │──► E2B 集群 + 平台代构建的镜像（每个 skill 版本一次）                │
                         │                                                                     │
                         │   动态注册 { (skill_name, version) → (engine, template_id) }       │
                         │   入 Module B registry，与 skill 版本一一绑定                       │
                         └─────────────────────────────────────────────────────────────────────┘
```

**关键约定**：

1. **协议（本文档）只定义左侧**：`runtime` 取值 + `.<engine>.dockerfile` 文件存在性。
2. **无全局 active-engine 配置**：平台**同时**运行 CubeSandbox + E2B 两个集群（v1 CubeSandbox 主、E2B 备 —— 但两个都常驻在线，作者按 `runtime` 取值路由）；运维不需要、也不能配置"全局 active engine 切换"。
3. **路由由 `runtime` 取值决定**：`cubesandbox::*` / `cubesandbox` → 路由到 CubeSandbox 集群；`e2b::*` / `e2b` → 路由到 E2B 集群。
4. **C1 落地内容**：
   - **Tier 1 镜像表**：维护 4 个 `runtime_value → template_id` 静态映射，配套 CubeSandbox（`cubemastercli tpl create-from-image`）与 E2B（`e2b template build`）各 2 份镜像构建脚本。
   - **Tier 2 构建管线**：作者上传 skill（含对应 `.<engine>.dockerfile`）→ 平台按 `runtime` 取值选目标引擎 → image build → push → `tpl create-from-image` / `template build` → 拿 `template_id` → 入 registry。
5. **集群级降级语义**：CubeSandbox 集群整体不可用时，`cubesandbox::*` / `cubesandbox` 路由失败并明确报错（`engine unavailable`）；作者可在 SKILL 中切换 runtime 到 `e2b::*` / `e2b` 应急。**v1 不做自动转译/路由 fallback**（同 §3.4 跨引擎"否"语义），避免"看似工作了实际行为变了"。

### 4.3 与 `e2b-code-interpreter` SDK 的兼容关系

CubeSandbox 是 E2B SDK drop-in：

```python
# 唯一差异：E2B_API_URL 与 template_id
# CubeSandbox：E2B_API_URL=http://cube-cluster:3000, template=cube-tpl-xxx
# E2B：        E2B_API_URL=（默认 e2b.dev）, template=python-basic

from e2b_code_interpreter import Sandbox
with Sandbox.create(template=template_id, envs=resolved_skillenv) as sb:
    result = sb.run_code(skill_entry_script)
```

→ 模块 C 的 `SandboxRuntime` 抽象只需做 **`runtime` → `(api_url, template_id)`** 的映射；SDK 调用面统一。无需为两个引擎各写一份执行器。

## 5. v1 不引入的枚举值及理由

| 候选 | 不引入理由 |
| --- | --- |
| `shell` / `bash` | 「能运行任意 shell」与「Python + `subprocess`」语义高度重叠，且 shell 沙箱极易变形为「自带运行时」（作者自装任意语言）—— 这种诉求恰好由 Tier 2 BYO `cubesandbox` / `e2b` 干净承接，无需独立 enum 值。 |
| `mcp-server` | MCP server 在沙箱内长驻 + 跨进程通信是另一种交互模型（不是「跑完一次脚本」），对生命周期 / 端口暴露 / 鉴权有不同要求；待 MCP 接入主线规划清楚后再表达，v1 不引入。 |
| `python-ml` / `python-pdf` / `python-browser` 等专用预制 | 镜像体积巨大（PyTorch 数 GB、OCR/Browser 各几百 MB）；冷启动 / 单机密度收益反比；且这类专用需求**正好该走 Tier 2 BYO**（作者最清楚自己要哪个版本），无需 Tier 1 提前承担。 |
| `firecracker` / `daytona` / 其他第三方引擎绑定值 | 当前只规划 CubeSandbox + E2B 两引擎，未来若加新引擎，再扩 Tier 2 取值即可，不在 v1 预留。 |
| `python-data` 独立预制 | 已并入 `python` —— 数据栈（pandas/numpy/matplotlib/openpyxl/python-docx/pillow）作为单一 `python` 的内置依赖，与 CubeSandbox `sandbox-code` / E2B `code-interpreter` 行业默认一致；warm-template 抹平镜像体积差异，无需独立 enum。 |
| Tier 1 上的"可选 layered Dockerfile" 中间层 | 即"`runtime: cubesandbox::python` + 可选 `.cubesandbox.dockerfile` 做增量"。**v1 不引入**，避免「同一 runtime 取值下，加载行为分两种」的歧义；需要叠加依赖的作者走 Tier 2 BYO，把基础镜像 `FROM` 我们的 Tier 1 镜像即可（C1 应保证 Tier 1 基础镜像可对外引用）。 |
| runtime 取值列表（如 `runtime: [cubesandbox, e2b]`） | §1 第 8 条预留；v1 单值，跨引擎可移植性靠作者改 `runtime` 前缀达成。 |

## 6. 与其他 A 子任务的同步

### 6.1 与 A1（frontmatter 字段）对齐

* **默认值**：A1 §2.3 给出 `runtime` 默认 `prompt-only` —— 本文档沿用，与 §2 表一致。
* **A1 §2.3 例值**：A1 写「如 `prompt-only` / `python-sandbox` / `node-sandbox`」。本文档定稿为引擎前缀模型 7 值（Tier 0: `prompt-only`；Tier 1: `cubesandbox::python` / `cubesandbox::node` / `e2b::python` / `e2b::node`；Tier 2: `cubesandbox` / `e2b`）。**建议 A1 §2.3 同步更新示例**（详见 §9）。
* **A1 §4.3 表述**：A1 §4.3 说「`runtime` 决定执行容器/E2B 模板」 —— 此处 "E2B 模板" 应宽化为 "执行模板，按 `runtime` 取值路由到对应引擎"。**建议 A1 §4.3 同步更新**（详见 §9）。

### 6.2 与 A2（`.skillenv`）对齐

* **`prompt-only` SKILL 的 `.skillenv`**：本文档 §3.1 规定 `prompt-only` 配 `.skillenv` 由加载器报错。A2 §2 表述「`.skillenv` 可选；无文件即跳过解析」，此规则不冲突 —— A2 是「文件不存在 → 不解析」，本文档加一条「文件存在但 runtime 是 `prompt-only` → 报错」。
* **vault 解析与 Tier 1 / Tier 2 / 引擎均无关**：vault 客户端（[TFRS-196](https://turingfocus.atlassian.net/browse/TFRS-196)）在 TFRobotServer 主进程内执行，解析后的明文通过 SDK `envs=...` 传入沙箱，与是 Tier 1 预制模板还是 Tier 2 BYO 镜像、是 CubeSandbox 还是 E2B 都无关。
* **Tier 2 镜像 `ENV` vs `.skillenv` 同名冲突**：Dockerfile 中的 `ENV KEY=value` 与 `.skillenv` 中 `KEY=...` 同名时，**`.skillenv` 通过 SDK `envs=...` 注入的值覆盖** —— 因为 SDK 注入发生在容器启动期、晚于 Dockerfile `ENV` 生效；这种顺序与 Docker / Linux env 语义一致。作者应避免在 Dockerfile 写入密钥类 `ENV`（密钥仍走 `.skillenv` + vault）。

### 6.3 与 A4（占位符）对齐

* **挂载点占位符**：A4（[TFRS-186](https://turingfocus.atlassian.net/browse/TFRS-186)）会定 `$TFROBOT_SKILL_DIR` / `$TFROBOT_API_ENDPOINT` / `$TFROBOT_SESSION_ID` 等占位符。本文档承诺这些占位符在所有沙箱化 runtime（Tier 1 的 `<engine>::python` / `<engine>::node` 与 Tier 2 的 `cubesandbox` / `e2b`）下**统一可用**；`prompt-only` 下无运行时上下文，占位符 N/A。

### 6.4 `.cubesandbox.dockerfile` / `.e2b.dockerfile` 文件契约

| 项 | 值 |
| --- | --- |
| 文件名 | 与 `runtime` 取值（Tier 2 的 `<engine>`）严格对应：`runtime: cubesandbox` → `.cubesandbox.dockerfile`；`runtime: e2b` → `.e2b.dockerfile`。两者都是 hidden-file 前缀，避免与作者本地 `Dockerfile` 冲突；文件名内含引擎信号，对应 §2.4 强契约。 |
| 位置 | skill 包根目录，与 `SKILL.md` 同级 |
| 是否必填 | 由 §2.4 强契约决定：`runtime: cubesandbox` ⇔ 必有 `.cubesandbox.dockerfile`、必无 `.e2b.dockerfile`；`runtime: e2b` ⇔ 反之；Tier 0/1 取值下 ⇔ 两个都不允许 |
| 文件格式 | 标准 [OCI Dockerfile](https://github.com/opencontainers/image-spec)；目标引擎规范见 §2.3 表 |
| 构建上下文 | 默认 = skill 包根目录（所以 `COPY scripts/foo.py /app/` 写法对作者直觉）|
| LLM 可见性 | **不进入 LLM 不可见黑名单**（与 `.skillenv` 区别）。理由：Dockerfile 不承载密钥（密钥统一走 `.skillenv`），LLM 可读 SKILL.md 与 scripts/，让它也看 Dockerfile 不增风险；review 阶段反而有用。 |
| 大小限制 | v1 不强约束；C1 实施时按构建主机资源给上限（建议 ≤ 256 KB Dockerfile + 单次构建产物 ≤ 5 GB），细节由 C1 定稿。 |
| 是否进入沙箱 FS | **不进入**。Dockerfile 是平台构建期消费的元数据，与 `.skillenv` 一样，运行时沙箱内不可见原文件（作者要拷贝某些路径到镜像，要么直接在 Dockerfile 里 `COPY`、要么放进 `scripts/`/`references/` 等 A4 标准目录由 runtime 挂载）。 |
| 同包同时含两文件 | **v1 不允许**（违反 §2.4 强契约即报错）。预留未来 `runtime: [cubesandbox, e2b]` 列表语法时可同时包含两份文件、平台按可用性挑（§1 第 8 条预留），v1 不实现。 |

**与 Module B（[TFRS-181](https://turingfocus.atlassian.net/browse/TFRS-181)）的衔接**：

- skill 上传/版本切换时，B 检测到 `.<engine>.dockerfile` 存在 → 按文件名解析目标引擎 → 触发 C1 构建管线（路由到对应引擎的 builder + registry）→ 拿到 `template_id` 写回 registry，与 (skill_version, engine) 绑定；
- skill 删除时反向清理已构建的 template；
- 同一 skill 不同版本 = 不同 image hash = 不同 template_id，作者修 Dockerfile 自动触发重建；
- skill 修改 `runtime` 取值（如 `cubesandbox` → `e2b`），等同于换了引擎，要求作者同步把 `.cubesandbox.dockerfile` 改名为 `.e2b.dockerfile` 并按目标引擎规范调整内容；旧 template 视为遗留版本，按版本生命周期处理。

## 7. 完整示例

### 7.1 极简 `prompt-only`

```yaml
---
name: claude-second-opinion
description: 让独立的 Claude 模型对当前会话给出第二意见。当用户希望获得另一个模型的独立视角时触发。
---

将当前会话最后一条用户消息原样发给 ……
```

（无 `runtime` → 默认 `prompt-only`；无 `.skillenv`）

### 7.2 `cubesandbox::python`（用户私人 token + 通用胶水栈）

```yaml
---
name: notion-page-summary
description: 抓取指定 Notion 页面并生成摘要。当用户给出 Notion 页面 URL 并要求"总结"时触发。
runtime: cubesandbox::python
compatibility: 需要用户在 Portal 注册 Notion 集成 token (vault key 名：NOTION_TOKEN)
---

运行 `$TFROBOT_SKILL_DIR/scripts/summarize.py`；页面 ID 由 LLM 通过 C4（[TFRS-197](https://turingfocus.atlassian.net/browse/TFRS-197)）定的输入注入机制传入 …
```

```dotenv
# .skillenv
NOTION_TOKEN=
```

> 想跑在 E2B 上：把 `runtime` 改成 `e2b::python` 即可，其他不变。

### 7.3 `cubesandbox::python`（数据分析）

```yaml
---
name: monthly-report
description: 从指定 Excel 月度账单生成 PDF 汇总报告并附图表。当用户上传 Excel 文件并要求月度报告时触发。
runtime: cubesandbox::python
compatibility: 需要 Excel 文件路径作为参数；输出 PDF 写入 $TFROBOT_SKILL_DIR/out/。pandas / openpyxl / matplotlib 已预装。
---

运行 scripts/build_report.py …
```

无 `.skillenv`（不依赖任何外部凭证）。

> `python` 预制已含数据栈（pandas / numpy / matplotlib / openpyxl / python-docx / pillow），不再细分 `python-data`。

### 7.4 `e2b::node`（TypeScript SKILL，调 Slack API）

```yaml
---
name: slack-thread-summarize
description: 抓取指定 Slack 线程并生成摘要。当用户给出 Slack 消息 URL 并要求"总结这个线程"时触发。
runtime: e2b::node
compatibility: 需要用户在 Portal 注册 Slack Bot OAuth token (vault key 名：SLACK_BOT_TOKEN)
---

运行 `$TFROBOT_SKILL_DIR/scripts/summarize.ts`（用 `npx tsx`）；thread URL 由 LLM 通过 C4（[TFRS-197](https://turingfocus.atlassian.net/browse/TFRS-197)）定的输入注入机制传入 …
```

> 同一 SKILL 想跑在 CubeSandbox：把 `runtime` 改成 `cubesandbox::node` 即可。

```dotenv
# .skillenv
SLACK_BOT_TOKEN=
```

沙箱内入口（`scripts/summarize.ts`）：

```typescript
import { z } from "zod";
const token = process.env.SLACK_BOT_TOKEN!;  // 来自用户 vault，由平台注入
// ... fetch + zod parse + 输出到 stdout
```

### 7.5 `cubesandbox` BYO（含 ffmpeg + 私有 wheel）

```yaml
---
name: video-thumbnail
description: 对上传的视频抽取关键帧并生成缩略图。当用户上传视频文件并要求"抽缩略图"时触发。
runtime: cubesandbox
compatibility: 自带镜像包含 ffmpeg 7.x 与私有内部库 turingfocus-media；构建期约 90 秒。
---

运行 scripts/thumb.py …
```

skill 包根 `.cubesandbox.dockerfile`：

```dockerfile
FROM ghcr.io/tencentcloud/cubesandbox-base:2026.16

RUN apt-get update \
    && apt-get install -y --no-install-recommends ffmpeg python3 python3-pip \
    && rm -rf /var/lib/apt/lists/*

# 私有 PyPI（构建机能访问）
RUN pip install --index-url https://pypi.turingfocus.internal/simple/ \
        turingfocus-media==1.4.2 pillow
```

`.skillenv` 仍可使用（解析与注入路径与 Tier 1 一致）：

```dotenv
# .skillenv
INTERNAL_S3_TOKEN=
```

> 想跑在 E2B：(1) `runtime` 改为 `e2b`；(2) 文件名改为 `.e2b.dockerfile`；(3) 内容改为符合 E2B template spec（base image 不同、不需要 envd）。强契约（§2.4）保证两文件不能同包共存。

### 7.6 引擎路由语义

| `runtime` 取值 | 路由目标 | 镜像来源 | CubeSandbox 集群挂时 | E2B 集群挂时 |
| --- | --- | --- | --- | --- |
| `prompt-only` | FastAPI 主进程内（无沙箱） | — | ✅ 不受影响 | ✅ 不受影响 |
| `cubesandbox::python` / `cubesandbox::node` | CubeSandbox 集群 | C1 平台预制模板 | ❌ 报 `engine unavailable` | ✅ 不受影响 |
| `e2b::python` / `e2b::node` | E2B 集群 | C1 平台预制模板 | ✅ 不受影响 | ❌ 报 `engine unavailable` |
| `cubesandbox` (BYO) | CubeSandbox 集群 | 本包 `.cubesandbox.dockerfile` 代构建产物 | ❌ 报 `engine unavailable` | ✅ 不受影响 |
| `e2b` (BYO) | E2B 集群 | 本包 `.e2b.dockerfile` 代构建产物 | ✅ 不受影响 | ❌ 报 `engine unavailable` |

→ 没有「全局 active engine」概念；每个 SKILL 的引擎归属由它自己的 `runtime` 取值显式声明，平台同时支持两套集群。要换引擎，**作者**改 `runtime` 前缀（以及 BYO 情况下连带改 Dockerfile 文件名 + 内容），不是平台运维改配置。

## 8. 准出对照

完成标准（来自 [TFRS-185](https://turingfocus.atlassian.net/browse/TFRS-185) 任务描述）：

* [x] runtime 枚举值列表确定 — §2 共 7 值（Tier 0: `prompt-only`；Tier 1: `cubesandbox::python` / `cubesandbox::node` / `e2b::python` / `e2b::node`；Tier 2: `cubesandbox` / `e2b`）
* [x] 每个值对应的执行容器/模板说明 — §3（作者可见契约）、§4（路由机制；具体 template_id 由 C1 定）
* [x] 与模块 C（[TFRS-194](https://turingfocus.atlassian.net/browse/TFRS-194) C1）的模板规范交叉确认 — §4.2 路由图 + §9.1 反溯 C1 范围扩到双集群常驻 + BYO 构建管线

**额外澄清**（任务描述未列但本设计必须显式表达）：

* [x] 双集群常驻模型（CubeSandbox + E2B 同时在线，按 `runtime` 取值路由，无全局 active-engine 配置） — §0、§4
* [x] `.cubesandbox.dockerfile` / `.e2b.dockerfile` 文件契约（per-engine 命名、位置、强契约、LLM 可见性、构建管线、不允许两文件同包共存） — §6.4
* [x] CubeSandbox 为 v1 主选 / E2B 为备选的工程依据 — §4.1
* [x] 引擎路由由 `runtime` 取值显式决定（不依赖运维配置） — §4.2、§7.6
* [x] `python` 单一预制 lang，不细分 `python-data` —— 与 CubeSandbox `sandbox-code` / E2B `code-interpreter` 行业默认一致 — §1 第 6 条、§2.5、§3.2

**AC 上限说明**：任务描述「2~3 个值」上限被本次设计突破到 7 值（Tier 0: 1 + Tier 1: 4 + Tier 2: 2），原因有二：① 引入了 BYO 自定义镜像能力（Tier 2），覆盖了原想用 Tier 1 表达却又 v1 不收的所有专用场景（`shell` / `mcp-server` / `python-ml` 等），无需新增专用 Tier 1 enum 承接长尾；② Tier 1 显式带引擎前缀（`cubesandbox::` / `e2b::`），使协议层 1:1 对应后端两套引擎集群常驻的实情，并让作者按需选择引擎。整体克制基调维持。详细考量见 §1 与 §5。

依赖下游子任务：

* Tier 1 镜像构建（CubeSandbox + E2B 双引擎各 2 份，共 4 份）+ Tier 2 BYO 构建管线（Buildkit/kaniko + push + `tpl create-from-image` / `template build` + registry 写回） → [TFRS-194](https://turingfocus.atlassian.net/browse/TFRS-194)（C1，**范围扩为双集群常驻 + BYO 管线**，见 §9.1）
* `.skillenv` 解析与注入实现 → [TFRS-195](https://turingfocus.atlassian.net/browse/TFRS-195)（C2）
* prompt-only 快速路径实现 → [TFRS-198](https://turingfocus.atlassian.net/browse/TFRS-198)（C5）
* `.cubesandbox.dockerfile` / `.e2b.dockerfile` 检测 + Tier 2 image build 触发 / template_id 与 (skill_version, engine) 绑定 → 模块 B [TFRS-181](https://turingfocus.atlassian.net/browse/TFRS-181) + C1

## 9. 任务描述 AC 修订建议（反溯）

A3 设计稿过程中发现以下上游 Jira / 已发布 A1 文档的表述与本文档校准后的设计不一致，建议视为已澄清/废弃，由 Story owner（[TFRS-180](https://turingfocus.atlassian.net/browse/TFRS-180)）决定是否直接修订字段：

### 9.1 模块 C 范围扩为双集群常驻 + BYO 构建管线

| Issue | 原表述 | 建议修订 |
| --- | --- | --- |
| [TFRS-185](https://turingfocus.atlassian.net/browse/TFRS-185)（本任务）标题 | `[A3] runtime 枚举值与 **E2B 模板** 的对齐` | `[A3] runtime 枚举值定稿（双引擎并存 + 引擎前缀语法 + BYO）` |
| [TFRS-185](https://turingfocus.atlassian.net/browse/TFRS-185) AC | "与模块 C（TFRS-182 C1）的 E2B 模板规范交叉确认" | "与 C1（[TFRS-194](https://turingfocus.atlassian.net/browse/TFRS-194)）的 **Sandbox 双集群常驻模板 + BYO 构建管线** 规范交叉确认" |
| [TFRS-194](https://turingfocus.atlassian.net/browse/TFRS-194) 标题 | `[C1] E2B 基础模板规范与版本管理` | `[C1] Sandbox 模板与镜像构建管线（双集群常驻 + BYO）` |
| [TFRS-194](https://turingfocus.atlassian.net/browse/TFRS-194) 任务描述 | "定义平台维护的 E2B 基础镜像模板...**SKILL 作者只能选择模板，不能自带 Dockerfile**" | "v1 同时常驻 CubeSandbox + E2B 两套集群（无全局 active-engine 配置），并同时提供 ① Tier 1 平台预制模板：每个 lang 在 CubeSandbox 与 E2B 各落地一份（`<engine>::python` / `<engine>::node`，共 4 份）；② Tier 2 BYO 构建管线：作者通过 `.cubesandbox.dockerfile` 或 `.e2b.dockerfile` 自带镜像，平台代为 build & push & 注册模板。**撤回原 AC「作者不能自带 Dockerfile」表述** —— BYO 是 v1 显式 escape hatch（详见 A3 §2.3 / §6.4）。" |
| [TFRS-194](https://turingfocus.atlassian.net/browse/TFRS-194) AC | "基础模板枚举（建议起步 `python-basic` 与 `python-data`）" | 拆为两组：① Tier 1 镜像枚举对齐 A3（`python` / `node` 两 lang × CubeSandbox + E2B 两引擎 = 4 份；`python` 含数据栈，不细分 `python-data`）；② Tier 2 BYO 构建管线设计（构建工具选型如 Buildkit/kaniko、平台 registry、镜像-skill-version-engine 绑定、清理策略）|
| [TFRS-182](https://turingfocus.atlassian.net/browse/TFRS-182) 标题 | `[SKILL][执行] 模块 C：**E2B** 执行引擎与环境变量注入` | `[SKILL][执行] 模块 C：Sandbox 双集群常驻 + 环境变量注入（CubeSandbox 主 / E2B 备 + BYO）` |
| [TFRS-182](https://turingfocus.atlassian.net/browse/TFRS-182) 任务描述 | "E2B 沙箱是首选方案——平台维护有限的几个基础模板" | "Sandbox（v1 主选 CubeSandbox / 备选 E2B，**两套集群同时常驻**，按 `runtime` 取值路由）是首选方案——平台维护少量预制模板（Tier 1），并提供 BYO Dockerfile 构建管线（Tier 2）支持极端场景" |
| [TFRS-182](https://turingfocus.atlassian.net/browse/TFRS-182) AC | "E2B 基础模板枚举与镜像内容确定" | "Sandbox 基础模板枚举（双引擎各一份）与镜像内容确定；Tier 2 BYO 构建管线方案确定；双集群同时常驻的运维配置确定" |
| [TFRS-199](https://turingfocus.atlassian.net/browse/TFRS-199) 任务描述 | "在 E2B 启动时挂载 SKILL 包目录、注入 ... 占位符" | "在 Sandbox 启动时（按 `runtime` 取值路由到 CubeSandbox 或 E2B；Tier 1 预制模板 / Tier 2 自建镜像）挂载 ..." |
| [TFRS-181](https://turingfocus.atlassian.net/browse/TFRS-181)（B 存储）AC | （v1 未显式提 Dockerfile 处理） | **新增**：「skill 包根 `.cubesandbox.dockerfile` / `.e2b.dockerfile` 检测（强契约校验：runtime 取值与文件名一一对应、不允许同包两文件共存）、版本绑定（image hash ↔ (skill_version, engine)）、上传时触发 C1 构建管线、删除时反向清理」|

### 9.2 A1 文档同步小幅修订

| 位置 | 原表述 | 建议修订 |
| --- | --- | --- |
| A1 §2.3 `runtime` 字段示例 | "如 `prompt-only` / `python-sandbox` / `node-sandbox`" | "Tier 0: `prompt-only`；Tier 1（平台预制，带引擎前缀）：`cubesandbox::python` / `cubesandbox::node` / `e2b::python` / `e2b::node`；Tier 2（BYO 引擎绑定）：`cubesandbox`（需配 `.cubesandbox.dockerfile`）/ `e2b`（需配 `.e2b.dockerfile`）— A3 定稿" |
| A1 §3.2 `network` 行 | "由 `runtime` 选中的 **E2B 模板** 隐式决定" | "由 `runtime` 选中的执行模板（按取值路由到 CubeSandbox 或 E2B 集群）隐式决定；Tier 2 BYO 时由作者在 `.<engine>.dockerfile` 中自决" |
| A1 §3.1 `shell` 不引入理由 | "E2B 沙箱固定 Linux，强制 bash" | "沙箱固定 Linux，强制 bash" |
| A1 §4.3 `runtime` 节 | "TFRobotServer 需要 `runtime` 指示模块 C 选择 **E2B 模板**" | "TFRobotServer 需要 `runtime` 取值显式指示目标引擎与模板（Tier 1 `<engine>::<lang>` = 平台预制；Tier 2 `<engine>` = 平台为 skill 构建的 BYO 镜像）" |
| A1 §4.3 例值 | "`python-sandbox` / `node-sandbox` → 对应 **E2B 镜像**" | "`cubesandbox::python` / `cubesandbox::node` / `e2b::python` / `e2b::node` → 对应模块 C 在该引擎上维护的预制模板；`cubesandbox` / `e2b` → 对应模块 C 为该 skill 构建的 BYO 镜像（需配同名 `.<engine>.dockerfile`）" |
| A1 §5.3 sandboxed 示例 | `runtime: python-sandbox` | `runtime: cubesandbox::python`（与 A3 定稿对齐）|
| A1 §3.2 v1 不引入 candidate 列表 | （未含 BYO Dockerfile 相关项） | **新增**：BYO Dockerfile 通过 `.cubesandbox.dockerfile` / `.e2b.dockerfile` 文件 + Tier 2 runtime 取值表达，**不在 frontmatter 内**（A1 §2.3 仅声明 enum 取值；文件契约见 A3 §6.4）|

→ A1 文档已合入 develop（[#35](https://cnb.cool/turingfocus/tfrobotv2/TFRobotServer/-/pulls/35)）。本次 A3 PR 一并提交上述 A1 小幅修订；commit message 中显式声明是 A3 反溯修订，便于追溯。

**设计教训 — 协议层与后端的对应**：A3 早期版本曾尝试"协议层完全引擎中立 + 后端二维映射 + 运维一项配置切换"模型；推进中发现这与"两套引擎集群同时常驻、不同 SKILL 对引擎诉求不同"的实情不匹配。最终模型把引擎名显式编码进 runtime 取值（`<engine>::<lang>` / `<engine>`）、Dockerfile 文件名（`.<engine>.dockerfile`）、强契约（runtime ↔ 文件名一一对应），让协议层与后端 1:1 对应、作者明确表达意图。代价是协议层多 2× 重复（每个 lang 出现两次），收益是没有「看似工作了实际换了引擎」的悬念。后续 Module A 类任务可借鉴这种"显式 > 暗中切换"原则。
