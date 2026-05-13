# A2 — SKILL 环境变量声明文件 `.skillenv` 设计

> Jira：[TFRS-184](https://turingfocus.atlassian.net/browse/TFRS-184)（Story [TFRS-180](https://turingfocus.atlassian.net/browse/TFRS-180) / Epic [TFRS-179](https://turingfocus.atlassian.net/browse/TFRS-179)）
> 范围：定义 `.skillenv` 文件的命名、位置、格式、解析规则与 LLM 不可见性保障；**用户凭证 vault** 的具体实现属 [TFRS-196](https://turingfocus.atlassian.net/browse/TFRS-196)（C3）需重新设计，[TFRS-195](https://turingfocus.atlassian.net/browse/TFRS-195)（C2）承接注入链路。
> 结论一句话：**`.skillenv` 是 skill 包根的标准 dotenv 文件，承载两类语义 —— 非空 VALUE 为字面量、空 VALUE 触发从用户私人凭证 vault 按 KEY 同名查询；ManagedLLM keystore 在协议层就被强制排除在外，物理上不存在被 SKILL 引用的可能**。

## 0. 重要前置澄清：与 ManagedLLM keystore 是不同的信任类别

任务描述（[TFRS-184](https://turingfocus.atlassian.net/browse/TFRS-184) / [TFRS-180](https://turingfocus.atlassian.net/browse/TFRS-180) / [TFRS-182](https://turingfocus.atlassian.net/browse/TFRS-182) / [TFRS-196](https://turingfocus.atlassian.net/browse/TFRS-196)）原表述里多次出现"复用 ManagedLLM 密钥库 / 模式"。在写本文档过程中发现该前提**不成立且存在安全风险**：

| 维度 | ManagedLLM keystore | `.skillenv` 注入目标 |
| --- | --- | --- |
| 谁出钱 / 所有权 | **平台**（OpenAI/Anthropic 等 quota 平台统一买） | **用户**（用户自己在 Portal 注册的三方 token） |
| 跑在哪段代码 | TFRobotServer 主进程内 `ManagedKeyClient.get_api_key` → 直接拼给 LLM SDK 调用，**全程在平台可信代码** | E2B sandbox 内**用户自己写的 SKILL 脚本**（用户可读 `os.environ`） |
| 跨平台-用户边界？ | **绝不**（key 从不返回给用户代码或视图） | **必然**（注入目的就是让用户脚本读到） |

→ 一旦把 ManagedLLM keystore 内容注入 `.skillenv`，**用户脚本一句 `print(os.environ)` 就能 dump 出平台付费 quota**，自存自用 / 跨账户复用 / 公网泄漏全无防护。这是从根上违反 ManagedLLM 信任前提的设计漏洞。

**本文档据此重新校准**：`.skillenv` 服务的**只能是**用户私人凭证 vault —— 用户自己注册的、所有权归用户、注入到用户自己的代码里。ManagedLLM keystore 在协议层就被排除，物理上无任何语法能引用它。详细边界划定见 §6；受此影响需要修订的上游 AC 见 §11。

## 1. 裁定原则

1. **服务对象唯一**：`.skillenv` 只服务"用户私人凭证 vault"。所有权和注入目的都在用户侧，没有跨信任边界的注入路径。平台付费的共享资源（如 ManagedLLM keystore）**不接入**本文件。
2. **最小可行（v1 克制）**：单文件 + 标准 dotenv 语法。不引入自定义前缀、变量插值、嵌套结构等扩展。
3. **沿用现有标准（dotenv）**：使用 `KEY=VALUE` 行格式，可用 python-dotenv / node dotenv 等现成解析器开箱解析，不造方言。
4. **双语义靠 dotenv 自身位差承载**：非空 VALUE → 字面量；空 VALUE → 用户 vault 按 KEY 同名查询。语法 100% 标准 dotenv，约定只在语义层。
5. **LLM 全程不可见**：文件 + 解析后值都不进入任何 prompt 渲染路径；E2B sandbox 内通过 `os.environ` 读取已解析的明文值，原文件不分发进沙箱。
6. **对接点而非实现**：本文档定稿语法与契约，resolver / vault 后端的具体实现属 [TFRS-195](https://turingfocus.atlassian.net/browse/TFRS-195)（C2 注入链路）+ [TFRS-196](https://turingfocus.atlassian.net/browse/TFRS-196)（C3 用户 vault 设计，需重新框定）。

## 2. 文件名与位置

| 项 | 值 |
| --- | --- |
| 文件名 | `.skillenv` |
| 位置 | **skill 包根目录**，与 `SKILL.md` 同级 |
| 是否必填 | 可选。无 `.skillenv` → skill 不声明任何环境变量需求，启动时跳过解析阶段 |
| 多文件 | **不支持**。v1 单文件即可；多环境（dev/prod）由 robot 配置层在调用前决定走哪个 vault 视图，而非由作者维护 `.skillenv.dev` / `.skillenv.prod` |

**为什么是 `.skillenv` 而非 `skill.env` / `secrets.yml`**：
- `.` 前缀属 Unix hidden-file 惯例（同档：`.gitignore` / `.editorconfig` / `.env`），与 dotenv 工具链生态一致。
- 与 `.env` 命名相近但避免冲突 —— 作者本地开发若在 skill 仓库内也用了 `.env`（如本地测试覆盖），二者不打架。
- 单文件、根目录唯一定位，加载器无需扫目录树，定位成本 O(1)。

## 3. 文件格式

选 **dotenv**（每行 `KEY=VALUE`，`#` 行注释，空行允许）。

### 3.1 候选格式对比

| 格式 | 解析器 | 与 E2B 注入接口契合度 | v1 元信息需求？ | 作者直觉 | 结论 |
| --- | --- | --- | --- | --- | --- |
| **dotenv** | python-dotenv / node dotenv 等开箱 | E2B SDK 的 `envs={...}` 本就是 `dict[str, str]`，零转换 | 无 | 写过本地 `.env` 的工程师全部都懂 | ✅ **采纳** |
| YAML | PyYAML / js-yaml | 需多一层反序列化 → 拍平为 `dict[str, str]` | 无（v1 不表达 required/default/description） | 引入「字符串引号是否必需」「list / dict 怎么写」等无关问题 | ❌ 过度结构化 |
| JSON | 内置 | 同 YAML，且不支持注释 | 无 | 不支持注释 → 作者难以批注 | ❌ 无注释 |

**退路明确**：若未来需要表达元信息（如 required / default / 多环境），届时再切 YAML 或显式 schema。本次只承诺 v1。

### 3.2 字符集与行长

- 编码：UTF-8（无 BOM）。
- 单行长度：建议 ≤ 256 字符；具体硬上限由 [TFRS-195](https://turingfocus.atlassian.net/browse/TFRS-195)（C2 注入器）按 E2B 模板限制定稿。
- 换行：LF。

## 4. 解析规则

```dotenv
# .skillenv ─ 两种语义都靠标准 dotenv 表达：
#   非空 VALUE → 字面量（沙箱 env 取该值）
#   空 VALUE   → 从用户 vault 按 KEY 同名查询

# 字面量：用户写 SKILL 时顺手的脚本常量
LOG_LEVEL=INFO
TIMEOUT_SECONDS=30

# 用户 vault 引用：vault 中 key 名 = 沙箱 env 名（v1 不支持 rename）
NOTION_TOKEN=
SLACK_BOT_TOKEN=
GITHUB_PAT=
```

### 4.1 行格式

| 规则 | 内容 |
| --- | --- |
| **R1** 行格式 | `KEY=VALUE`。KEY 形如 `[A-Z_][A-Z0-9_]*`（POSIX env var 名约定）。 |
| **R2** VALUE 非空 → **字面量** | 注入器把 VALUE 整段当作 env var 值传给 E2B；不做任何解析、变量替换。沙箱内 `os.environ[KEY] == VALUE`。 |
| **R3** VALUE 为空 → **用户 vault 同名查询** | 注入器以 KEY 为名查询用户私人凭证 vault；查到 → 注入解析后的明文；查不到 → 启动失败。 |
| **R4** 注释 | `#` 起始的行为注释；**行尾注释不支持**（避免 VALUE 含 `#` 时的转义复杂度）。 |
| **R5** 空行 | 允许。 |
| **R6** 引号 / 变量插值 | **不支持** `"..."` / `'...'` 包裹，**不支持** `${VAR}` / `$VAR` 展开。VALUE 始终被当作完整字符串字面量。 |
| **R7** KEY 重复 | **解析报错**。同一 KEY 多次出现即拒绝，避免覆盖语义混淆。 |
| **R8** 多行 VALUE | **不支持**。每个声明必须单行。 |

### 4.2 字面量与 vault 引用的"零自定义语法"消歧

dotenv 标准本身允许 `KEY=`（空值）和 `KEY=value`（非空值）两种合法行。我们**没有引入任何 dotenv 标准之外的语法**，只是在语义层约定：

- 非空 → 字面量传递（这就是 dotenv 标准语义）
- 空 → 触发外部 vault 查询（这是本协议在 dotenv 语义之外加的**唯一**一条约定）

**为什么不用 `@platform:` / `@vault:` 类前缀消歧**：
- 这类前缀不是 dotenv 标准 —— 现成解析器把 `@platform:openai` 当纯字符串，引入意味着自造解析器，工具链失配。
- 空 VALUE 在 dotenv 中虽然标准上代表"空字符串"，但**实际项目中作者写 `KEY=` 且明确意图就是空字符串的场景极罕见**（要表达常量空字符串通常会写 `KEY=""` 或脚本里硬编码）。把这个边角语义重定义为 "vault 查询" 性价比高。
- 副作用：**vault 中的 key 名被强制等于 env var 名（v1 不支持 rename）** —— 这反而是好的约束：迫使 vault 命名向 OS/库的 env var 习惯对齐（如 `NOTION_TOKEN` 而非 `notion_token`），命名一致性提高。

**为什么不支持 `KEY=""` 字面量空字符串**：v1 不区分 `KEY=`（vault 查询）和 `KEY=""`（字面量空字符串）—— 都视为 vault 查询。需要常量空字符串的极罕见场景由作者在脚本里手动 `os.environ[K] = ""` 处理；不让 `.skillenv` 为这类 1% 的需求引入引号语法。

### 4.3 错误契约（初步）

| 情形 | 行为 |
| --- | --- |
| 文件不存在 | 跳过解析，env 注入集合 = {}，正常启动 |
| 解析错误（R1/R4/R6/R7/R8 违反） | 启动失败，报错指明行号与违反规则 |
| 空 VALUE 行（R3） + vault 查不到对应 key | 启动失败，**只暴露 KEY 名**给作者/日志（不暴露 vault 内部错误细节，避免 secret 命名空间侧信道） |
| 同进程并发解析 | 由调用方串行化，本文档不约束 |

具体错误码、缺失时是否允许 fallback、并发处理细节等由 [TFRS-195](https://turingfocus.atlassian.net/browse/TFRS-195)（C2）定稿。

## 5. LLM 不可见性保障（三层防护）

`.skillenv` 中的内容（文件本体 + 解析后的值）**必须**不进入任何 LLM 可见的渲染路径。下列三层防护**全部**实施，缺一不可：

### 5.1 加载层黑名单（模块 B）

SKILL 加载器维护显式黑名单：

```
_LLM_INVISIBLE_FILES = {".skillenv"}
```

无论 LLM 通过何种路径请求该文件（progressive disclosure 自主读、工具调用读、相对路径解析、`allowed-tools` 中的文件读工具），加载器都**拒绝返回内容**并记录审计日志（含 robot_id / tenant_id / skill_name / attempted_path）。

**实施位置**：模块 B（[TFRS-181](https://turingfocus.atlassian.net/browse/TFRS-181)）SKILL 加载器。
**测试责任**：模块 B 单元测试 + 集成测试覆盖「LLM 请求 `.skillenv` 路径 → 加载器拒绝」用例。

### 5.2 沙箱注入而非文件分发（模块 C）

E2B 启动器在 `sandbox.create()` 前完成 `.skillenv` 解析，把结果以 `envs={...}` 形式注入容器进程环境；**原 `.skillenv` 文件本体不进入沙箱 FS**。

作者脚本通过 `os.environ["NOTION_TOKEN"]` 读取明文值；脚本中**看不到 vault 元信息**（解析后只有「已解析的凭证明文」存在于进程环境中，"这条来自字面量 vs 来自 vault" 等元信息只在 TFRobotServer 主进程中存活，从未跨进程边界）。

**实施位置**：[TFRS-195](https://turingfocus.atlassian.net/browse/TFRS-195)（C2 执行侧注入链路）。
**测试责任**：C2 集成测试覆盖「`.skillenv` 不复制到沙箱 FS」「`envs` 注入成功」两条用例。

### 5.3 prompt 渲染产物 grep 守护（CI 黑测试）

模块 B 加载测试中加 CI 黑测试 —— 给定一个带 `.skillenv` 的样例 skill 包，断言**最终拼到 LLM 的 prompt 字符串中不包含 `.skillenv` 文件内的任意一行内容**（包括 KEY 名、VALUE 字面量、注释）。

防止任何「加载器代码后续重构遗漏黑名单」的退化 —— 即使有人不慎打破 5.1 的拒绝逻辑，本测试会捕获。

**实施位置**：模块 B 测试套件。
**测试责任**：CI 强制阻塞测试。

## 6. 与平台密钥基础设施的边界

### 6.1 强制隔离：ManagedLLM keystore **不进入** `.skillenv` 视野

| 项 | 边界 |
| --- | --- |
| 协议层语法 | `.skillenv` 没有任何语法能引用 ManagedLLM keystore（无 `@platform:`、无 namespace、无 keystore 类型选择）。物理上不可能写出"引用 OpenAI quota key"的行。 |
| 解析层映射 | 注入器收到空 VALUE 时**只查询用户 vault**，不查询 ManagedLLM keystore。即使 vault 中存在同名 key 与 ManagedLLM keystore 巧合同名，也只看 vault。 |
| 注入层目标 | E2B sandbox env vars 只包含用户 vault 解析结果 + 用户写的字面量。**ManagedLLM keystore 内容永远不出现在 sandbox env**。 |

→ 一句话：**`.skillenv` 协议层、解析层、注入层三处都强制隔离 ManagedLLM keystore**。即使有人误把 ManagedLLM 的 key 名（如 `openai`）写到 `.skillenv` 里，它也只会被当成"查 vault 里有没有叫 `openai` 的用户凭证"，**永远不会拿到 ManagedLLM 那张表里的平台 quota key**。

### 6.2 用户私人凭证 vault（待 TFRS-196 重新设计）

`.skillenv` 引用的对象是一个**新的、独立于 ManagedLLM 的存储类别**：用户在 Portal 上注册的、归属用户自己的第三方凭证（Notion / Slack / GitHub PAT / 自有 OpenAI key 等），平台以加密形式代为存储。

| 维度 | v1 初步契约 |
| --- | --- |
| 所有权 | 用户。平台为代管，用户随时可撤销 / 修改。 |
| 作用域 | 至少按 `(tenant_id, user_id)` 隔离；按 robot/skill 进一步约束（哪个 skill 可读哪些 key）由 [TFRS-196](https://turingfocus.atlassian.net/browse/TFRS-196) 决定。 |
| 查询接口 | 按 key 名查询（v1 不支持 rename，等同于按 env var 名查询）。 |
| 共享底座基础设施 | **允许**与 ManagedLLM 共享 TFRSManager 加密传输、RSA 私钥等**通用基础设施**。 |
| 共享 keystore 表 / 查询接口 | **禁止**。这是与 ManagedLLM 的核心隔离点。复用 `ManagedKeyClient.get_api_key()` 即等于打破隔离。 |

**待 [TFRS-196](https://turingfocus.atlassian.net/browse/TFRS-196) 定稿**：vault 后端表结构、Portal 注册流程、按 skill / robot 的访问控制粒度、错误码集合等。

### 6.3 与 ManagedLLM 仅共享什么

允许复用：
- TFRSManager 远端服务的**加密传输通道**（RSA 私钥 + 解密链路）
- `KeyCache` 一类**通用 TTL 缓存基础设施**
- DI 容器中的**注入模式**

禁止复用：
- ManagedLLM 的 **keystore 表**（不同存储类别，不同所有权）
- `ManagedKeyClient.get_api_key()` 接口本身（其调用方默认是平台可信代码；`.skillenv` resolver 不是）
- ManagedLLM 已注册的 provider_name 命名空间（`openai` / `anthropic` 等是平台公共标识符，用户 vault 命名应是用户私有命名空间）

## 7. 解析与注入链路（与 TFRS-195 / TFRS-196 的边界）

本文档**定稿**的是协议（语法 + 契约 + 不可见性约束 + 隔离边界），不定稿实现。

| 阶段 | 责任方 | 备注 |
| --- | --- | --- |
| ① 读取 `.skillenv` 文件 | [TFRS-195](https://turingfocus.atlassian.net/browse/TFRS-195) C2 启动器 | 文件路径：MinIO 对象存储中 skill 包的根对象 |
| ② 按 §4 规则解析为 `dict[KEY, (kind, raw_value)]`（kind ∈ {literal, vault\_lookup}） | C2 启动器 | 校验 R1–R8，失败即整体启动失败 |
| ③ 对 `vault_lookup` 项，通过 vault 客户端（[TFRS-196](https://turingfocus.atlassian.net/browse/TFRS-196) 提供）按 KEY 查询 | C2 启动器 | 不调 `ManagedKeyClient.get_api_key()`；调新的用户 vault 客户端（接口待 C3 定） |
| ④ 构造 `envs = {literal items} ∪ {resolved vault items}` 注入 `sandbox.create(envs=...)` | C2 启动器 | 原 `.skillenv` 文件本体不进入沙箱 FS（§5.2） |
| ⑤ 沙箱进程 `os.environ[KEY]` 读取 | 作者脚本 | 沙箱内已是明文值，无需再解析 |

## 8. v1 不引入的能力及理由

| 能力 | 不引入理由 |
| --- | --- |
| 来源标记前缀（`@platform:` / `@vault:` 等） | §4.2 已论证：空 VALUE 触发 vault 查询已是足够约定，零自定义语法即可消歧；引入前缀只会脱离 dotenv 标准。 |
| vault key 与 env var 名 rename | v1 强制同名（vault key = KEY）；rename 在多账户、密钥别名等场景才有需求，v1 不出现。如有需要，TFRS-196 评估时再决策语法演进。 |
| 多文件加载（`.skillenv.local` / `.skillenv.prod`） | v1 单文件即可；多环境由 robot 配置层在调用前切 vault 视图，而非作者维护多份。 |
| 变量插值（`URL=https://${HOST}:${PORT}`） | 拼接由作者在沙箱内脚本中完成（`${VAR}` 由 shell / Python 自然处理），不在 `.skillenv` 文件层做。 |
| 类型注解 / 校验规则 / required 标记 | 缺失即启动失败（§4.3），作者按报错修正即可，不需要单独标 required。 |
| 嵌套 / 命名空间（`db.host` / `db.port`） | 容器内 env var 本就是扁平命名空间。 |
| 行尾注释 | 避免 VALUE 含 `#` 时的转义复杂度；注释统一用整行 `# ...`。 |
| 字面量空字符串区分（`KEY=` vs `KEY=""`） | §4.2 已论证：将 `KEY=` 重定义为 vault 查询带来的收益大于"区分空字符串字面量"的损失。 |

## 9. 完整示例

### 9.1 极简（仅一个用户 vault 引用）

```dotenv
# .skillenv
# 该 SKILL 调 Notion API，需要当前用户的 Notion token
NOTION_TOKEN=
```

对应 SKILL.md：

```yaml
---
name: notion-page-summary
description: 抓取指定 Notion 页面并生成摘要。当用户给出 Notion 页面 URL 并要求"总结"时触发。
runtime: python-sandbox
compatibility: 需要用户在 Portal 注册 Notion 集成 token (vault key 名：NOTION_TOKEN)
---
```

沙箱内脚本：

```python
import os, requests
token = os.environ["NOTION_TOKEN"]  # 已解析明文，来自用户 vault
resp = requests.get("https://api.notion.com/v1/pages/...", headers={"Authorization": f"Bearer {token}"})
```

### 9.2 字面量 + 用户 vault 混合

```dotenv
# .skillenv

# 字面量常量：脚本日志等级 / 超时阈值
LOG_LEVEL=INFO
TIMEOUT_SECONDS=30

# 用户 vault 引用：跨多个三方服务的私人 token
SLACK_BOT_TOKEN=
GITHUB_PAT=
```

对应 SKILL.md：

```yaml
---
name: incident-triage
description: 把 GitHub Issue 同步到 Slack 频道并按内部分类标签处理。当用户要求设置"GitHub ↔ Slack 联动"时触发。
runtime: python-sandbox
compatibility: 需要用户在 Portal 注册 GitHub PAT 与 Slack Bot OAuth token
---
```

### 9.3 prompt-only skill 一般不需要 `.skillenv`

```yaml
# SKILL.md
---
name: claude-second-opinion
description: 在主模型回答完毕后，调一次 Claude 拿独立意见。
---
```

无 `.skillenv` 文件。原因：

- prompt-only skill 的"调用 LLM" 走的是 robot 已配置好的 LLM 通道（包括 ManagedLLM 那批平台付费 key），由平台主进程内的可信代码处理；
- skill 本身只是 prompt 编排，**不应该**也**不需要**自己声明 LLM API key。

如果某 prompt-only skill 真有访问"用户自己的 OpenAI 个人 key"的需求（如绕过平台 quota 用自己额度），那也属"用户自己的凭证"范畴，可以走 `.skillenv` 引用 vault，与 ManagedLLM 平台 quota 不冲突。

## 10. 准出对照

完成标准（来自 [TFRS-184](https://turingfocus.atlassian.net/browse/TFRS-184) 任务描述）：

- [x] 文件名、位置、文件格式（YAML / dotenv / JSON）确定 — §2、§3（dotenv，包根 `.skillenv`）
- [x] 密钥来源标记语法确定 — §4（标准 dotenv 双语义：非空 = 字面量，空 = 用户 vault 同名查询；显式拒绝 `@platform:` URI 前缀）
- [x] 与现有密钥基础设施的对接点 — §6.3（仅复用 TFRSManager 加密传输等底座，**禁止**复用 ManagedLLM keystore 表与查询接口）
- [x] LLM 不可见性的保障机制 — §5（三层防护：加载层黑名单 / 沙箱注入而非文件分发 / CI grep 守护）

**额外澄清**（任务描述未列但本设计必须显式表达）：

- [x] 与 ManagedLLM keystore 的强制隔离边界 — §0、§6.1（协议层 / 解析层 / 注入层三处隔离，物理不可引用）
- [x] 用户私人 vault 的存在性与契约草案 — §6.2（待 [TFRS-196](https://turingfocus.atlassian.net/browse/TFRS-196) 定稿，本文档约定边界）

依赖下游子任务：

- E2B 模板枚举与 env var 注入接口细节 → [TFRS-185](https://turingfocus.atlassian.net/browse/TFRS-185)（A3）+ [TFRS-195](https://turingfocus.atlassian.net/browse/TFRS-195)（C2）
- `.skillenv` resolver 实现、错误码、缺失处理 → [TFRS-195](https://turingfocus.atlassian.net/browse/TFRS-195)（C2）
- **用户 vault 后端设计**（表结构 / Portal 注册 / 访问控制 / 客户端接口） → [TFRS-196](https://turingfocus.atlassian.net/browse/TFRS-196)（C3，需重新框定，见 §11）
- 加载器 `_LLM_INVISIBLE_FILES` 黑名单实施与 grep 守护测试 → 模块 B [TFRS-181](https://turingfocus.atlassian.net/browse/TFRS-181)

## 11. 任务描述 AC 修订建议（反溯）

A2 设计稿过程中发现以下上游 Jira 描述的表述与本文档校准后的设计不一致，建议视为已澄清/废弃，由 Story owner 决定是否直接修订字段：

| Issue | 原表述 | 建议修订 |
| --- | --- | --- |
| [TFRS-184](https://turingfocus.atlassian.net/browse/TFRS-184) | "密钥来源标记语法确定（**建议 `KEY=@platform:secret_name`**）" | 改为「标准 dotenv 双语义」；前缀语法不引入（详见 §4） |
| [TFRS-184](https://turingfocus.atlassian.net/browse/TFRS-184) | "与 **ManagedLLM 现有密钥库**的对接点初步约定" | 改为「与用户私人 vault 的对接点；与 ManagedLLM keystore 强制隔离」（详见 §6） |
| [TFRS-182](https://turingfocus.atlassian.net/browse/TFRS-182) | "密钥…平台启动 E2B 时读取…从平台密钥库（**复用 ManagedLLM 模式**）注入" | 改为「从用户私人 vault 注入；与 ManagedLLM 仅共享 TFRSManager 加密传输等底座基础设施」 |
| [TFRS-196](https://turingfocus.atlassian.net/browse/TFRS-196) | 标题：「平台密钥托管对接（**复用 ManagedLLM 机制**）」 | 标题改为「用户凭证 vault 独立设计（与 ManagedLLM 信任隔离）」 |
| [TFRS-180](https://turingfocus.atlassian.net/browse/TFRS-180) | AC 中 "密钥来源标记法" 默认指向 ManagedLLM | 显式声明所指为用户 vault；ManagedLLM 不在视野内 |
| [TFRS-195](https://turingfocus.atlassian.net/browse/TFRS-195) | "从 .skillenv 到 E2B 容器的注入链路…查密钥库" | "密钥库"明确为用户 vault（[TFRS-196](https://turingfocus.atlassian.net/browse/TFRS-196) 提供），不调 `ManagedKeyClient.get_api_key()` |

**信任边界教训**：从 [TFRS-184](https://turingfocus.atlassian.net/browse/TFRS-184) 这次"差点把 ManagedLLM 平台付费 quota 注入用户代码"的设计漏洞看，任务描述里"复用 X 实现 Y"类表述在跨信任边界场景下需要格外警惕 —— X 的安全前提（"key 永不跨平台-用户边界"）在 Y 的本质需求（"注入用户代码"）下完全反转。后续 Epic 推进时建议把"信任/计费/所有权"三轴拉到 AC 评审清单的固定项。
