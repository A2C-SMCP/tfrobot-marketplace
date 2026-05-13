# TFRobotServer Marketplace v1 规范

> Jira：D1 [TFRS-202](https://turingfocus.atlassian.net/browse/TFRS-202) / Module D Story [TFRS-201](https://turingfocus.atlassian.net/browse/TFRS-201) / Epic [TFRS-179](https://turingfocus.atlassian.net/browse/TFRS-179)
> 范围：SKILL 包的 **Git 仓库分发约定（数据结构与 JSON Schema）**；与单个 SKILL 文件夹的内容契约（[A5 协议规范](../skill/protocol-v1.md)）正交
> 设计参考：[Claude Code Plugin Marketplaces 官方规范](https://code.claude.com/docs/en/plugin-marketplaces) —— 字段定义与 source 类型枚举借鉴其设计，**但 manifest 路径与命名空间独立**（`.tfrobot-plugin/` 而非 `.claude-plugin/`）
> 读者：① SKILL / Plugin 作者（通过 Marketplace 发布场景）② Module D 实施工程师

## 0. 总览

### 0.1 这是什么

**Marketplace** 是 SKILL/Plugin 的 **Git 仓库分发标准**。一个 Marketplace 是一个 Git 仓库（或可被 Git 操作的等价物），仓库里以 `.tfrobot-plugin/marketplace.json` 为入口列出该 catalog 提供的 plugin。

层级语义（自顶向下）：

* **Marketplace** = 一个 Git 仓库；**catalog 单位**；manifest 为 `.tfrobot-plugin/marketplace.json`
* **Plugin** = 一组相关能力的包装；**版本与依赖单位**；manifest 为 `.tfrobot-plugin/plugin.json`
* **SKILL** = 单个能力包；**内容单位**；由 [A5 协议规范](../skill/protocol-v1.md) 定义

### 0.2 与 SKILL 协议 (A5) 的关系

| 维度 | A5（SKILL 协议） | 本规范（Marketplace） |
| --- | --- | --- |
| 范围 | SKILL 文件夹的内容契约（frontmatter / 目录 / runtime / 占位符 / `skills` 工具等） | Git 仓库的目录约定 + JSON Schema |
| 作者关心 | 任何 SKILL 作者必须遵守 | 仅"通过 Marketplace 发布"的作者关心 |
| 平台关心 | LLM / Sandbox 加载 SKILL 时遵守 | Module D 从 Git 仓库拉取 plugin / SKILL 时遵守 |
| 正交性 | 两者**完全正交** —— 同一 SKILL 文件夹既可被 Portal 直接上传也可被 Marketplace 拉取，加载语义一致 | |

### 0.3 与 Claude Code Marketplace 的关系

**TFRobotServer Marketplace 是独立标准**，借鉴 Claude Code 的字段设计与 source 类型枚举，但 **manifest 路径（`.tfrobot-plugin/`）与 catalog 命名空间独立**，不追求"同一份仓库被两端互换消费"。

**P0（字段/Schema 借鉴）**：marketplace.json / plugin.json 字段名、必填可选契约、source 类型枚举与字段约束沿用 Claude Code 设计 —— 作者切换阵地时学习成本最低。

**P1（路径独立）**：manifest 路径采用 `.tfrobot-plugin/` 命名空间；同一份仓库要双端消费需作者维护两套 manifest 目录（双轨发布），平台不做隐式兼容。

**P2（SKILL 层正交）**：A5 引入的私有内容（`.cubesandbox.dockerfile` / `.e2b.dockerfile` / `.skillenv` 等）位于 SKILL 文件夹内部，不属于本规范范围。

## 1. 设计原则

1. **字段设计借鉴 Claude Code Plugin Marketplaces**：核心字段名、source 类型枚举、必填/可选契约沿用其设计，作者跨阵地切换成本最低；不发明替代命名。
2. **manifest 命名空间独立**：使用 `.tfrobot-plugin/` 而非 `.claude-plugin/`，避免目录名占用别家产品命名；同时不假装与 Claude Code 仓库结构互换消费。
3. **与 SKILL 协议 (A5) 正交**：本规范不修改单个 SKILL 文件夹的内容契约。
4. **manifest 显式分级**：`marketplace.json` 与 `plugin.json` 是两份不同的 manifest，位于不同目录层；不存在"marketplace 嵌套 marketplace"——只存在"marketplace 通过 source 引用 plugin"。
5. **Plugin source 是 plugin 级别的引用**：`git-subdir` 等 source 类型引用的 `path` 必须指向 plugin 边界（含 `plugin.json` 的目录），不可指向 marketplace 边界。
6. **非标准扩展显式标注**：任何超出参考标准的字段或 source type（如 `cnb`）必须在 §10 与 §11 中显式列出。

## 2. 仓库布局

### 2.1 Marketplace 仓库根

```
<Git 仓库根 = Marketplace>/
  .tfrobot-plugin/
    marketplace.json                          # 必需，Marketplace manifest（§3）
  plugins/                                    # 推荐路径；可被 metadata.pluginRoot 覆写
    <plugin-name-A>/
      .tfrobot-plugin/                        # 条件必需：见 §6 与 §4.4 strict mode
        plugin.json                           # strict=true 时必需；strict=false 时省略
      skills/
        <skill-name>/                         # SKILL 内容 —— A5 §2~§10
          SKILL.md
          .skillenv                           # 可选（A5 §5）
          .cubesandbox.dockerfile             # 条件必需（A5 §6.3）
          .e2b.dockerfile                     # 条件必需（A5 §6.3）
          scripts/
          references/
          assets/
    <plugin-name-B>/
      .tfrobot-plugin/plugin.json             # 同上，条件必需
      skills/...
```

### 2.2 路径约定

| 约定 | 值 |
| --- | --- |
| Marketplace manifest 路径 | `.tfrobot-plugin/marketplace.json`（仓库根，**必需**） |
| Plugin manifest 路径 | `<plugin>/.tfrobot-plugin/plugin.json`（**条件必需**，见 §6） |
| Plugin 默认聚集目录 | `plugins/<plugin-name>/`（可被 `metadata.pluginRoot` 覆写） |
| SKILL 目录 | `<plugin>/skills/<skill-name>/`（强制） |
| Plugin / SKILL 名称字符集 | `[a-z0-9-]`，kebab-case，不以 `-` 开头/结尾，无连续 `--` |

### 2.3 SKILL 引用

* **三元组**：`(marketplace_name, plugin_name, skill_name)`
* **串形式**：`marketplace_name/plugin_name/skill_name`
* **Plugin 安装语法（Claude Code 兼容）**：`/plugin install <plugin-name>@<marketplace-name>`

## 3. marketplace.json 字段规范

> 字段设计沿用 [Claude Code Marketplace Schema](https://code.claude.com/docs/en/plugin-marketplaces#marketplace-schema)；manifest 路径独立为 `.tfrobot-plugin/`。

### 3.1 必填字段

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `name` | string | Marketplace 标识（kebab-case，无空格）。公开可见；用户安装时显示：`/plugin install x@<name>` |
| `owner` | object | 维护者信息（见 §3.3） |
| `plugins` | array&lt;Plugin&gt; | Plugin 条目数组（见 §4） |

> **保留名**：TFRobotServer 官方将保留若干 marketplace 名称用于官方分发（如 `tfrobot-skills-official` / `tfrobot-marketplace` / `turingfocus-skills`）；第三方 marketplace 不得占用以 `tfrobot-` / `turingfocus-` / `tfs-` 开头的官方命名空间。Module D 实施时输出完整清单。

### 3.2 可选字段

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `$schema` | string | JSON Schema URL（编辑器自动补全用，运行时忽略） |
| `description` | string | 简要描述 |
| `version` | string | Marketplace manifest 版本 |
| `metadata` | object | 备用元数据容器；`description` / `version` 可放此字段下（向后兼容） |
| `metadata.pluginRoot` | string | 相对 plugin source 的基准目录前缀（如 `"./plugins"`，使 `"source": "formatter"` 等价于 `"source": "./plugins/formatter"`） |
| `allowCrossMarketplaceDependenciesOn` | array | 本 marketplace 中的 plugin 允许依赖的其他 marketplace 白名单 |

### 3.3 Owner 子对象

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `name` | string | 是 | 维护者或团队名称 |
| `email` | string | 否 | 联系邮箱 |

### 3.4 marketplace.json 完整示例

```json
{
  "$schema": "https://anthropic.com/claude-code/marketplace.schema.json",
  "name": "tfs-skills-official",
  "description": "Official TFRobotServer SKILL marketplace",
  "owner": {
    "name": "TFRobotServer Team",
    "email": "tfs@turingfocus.com"
  },
  "metadata": {
    "pluginRoot": "./plugins"
  },
  "plugins": [
    {
      "name": "data-toolkit",
      "source": "data-toolkit",
      "description": "Data ingestion / cleansing / report SKILLs",
      "version": "1.2.0",
      "author": { "name": "TFRobotServer Team" },
      "category": "data",
      "tags": ["etl", "pandas"]
    },
    {
      "name": "auth-tools",
      "source": {
        "source": "github",
        "repo": "turingfocus/tfs-auth-plugin",
        "ref": "v0.3.1"
      },
      "description": "Auth-related SKILLs"
    }
  ]
}
```

## 4. Plugin 条目字段规范

> Plugin 条目可包含 Plugin Manifest（§6）允许的所有字段，加上以下 marketplace 专属字段。

### 4.1 必填字段

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `name` | string | Plugin 标识（kebab-case，无空格） |
| `source` | string \| object | Plugin 拉取源（见 §5） |

### 4.2 可选 metadata 字段

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `description` | string | 简要描述 |
| `version` | string | Plugin 版本。设置后用户仅在该字符串变更时收到更新；省略则回退到 git commit SHA |
| `author` | object | `{name: string (必需), email?: string}` |
| `homepage` | string | 主页或文档 URL |
| `repository` | string | 源码仓库 URL |
| `license` | string | SPDX license（如 `MIT`、`Apache-2.0`） |
| `keywords` | array&lt;string&gt; | 用于发现与分类的关键字 |
| `category` | string | 类别 |
| `tags` | array&lt;string&gt; | 标签 |
| `strict` | boolean | 是否以 plugin.json 为组件定义权威（默认 `true`，见 §4.4） |

### 4.3 可选组件配置字段（marketplace 条目级覆写）

> 这些字段可在 marketplace 条目中**追加或覆写** plugin 内部 `plugin.json` 的同名字段；语义见 §4.4 strict mode。

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `skills` | string \| array | 自定义 SKILL 目录路径（含 `<name>/SKILL.md`） |
| `commands` | string \| array | 自定义命令 `.md` 文件或目录 |
| `agents` | string \| array | 自定义 agent 文件路径 |
| `hooks` | string \| object | Hooks 配置或 hooks 文件路径 |
| `mcpServers` | string \| object | MCP server 配置或路径 |
| `lspServers` | string \| object | LSP server 配置或路径 |

### 4.4 Strict mode

| `strict` 值 | 语义 |
| --- | --- |
| `true`（默认）| `plugin.json` 是组件定义权威；marketplace 条目可**补充**额外组件，二者合并 |
| `false` | marketplace 条目是组件定义全集；若 plugin 仓库自带 `plugin.json` 也声明组件，则装载失败 |

**何时用 `false`**：marketplace 运营方希望对 plugin 仓库做 curation —— 不修改上游仓库内容，但通过条目精选/重构暴露出来的组件子集。

## 5. Plugin source 字段规范

> 本节定义 **Plugin source** —— 即 `marketplace.json` 中每个 plugin 条目的 `source` 字段。这是与 **Marketplace source**（用户添加 marketplace 时用的 source）**完全独立的两套 schema**，详见 §5.4。

### 5.1 Source 类型枚举（5 种 Git 引用方式）

全部是 Git 仓库的引用，差别仅在简写糖归一化目标与"是否子目录"。Claude Code 还支持 `npm`/`pip` 包源，本项目不引入（§10 说明理由）。

| Source | 形态 | 字段 | 说明 |
| --- | --- | --- | --- |
| **相对路径** | `string`（如 `"./my-plugin"`）| 无 | marketplace 仓库内的 plugin 目录；必须以 `./` 开头；不允许 `..` |
| `url` | object | `url`, `ref?`, `sha?` | 任意 Git URL（`https://` / `git@`），含 GitLab / Bitbucket / 自建 |
| `github` | object | `repo`, `ref?`, `sha?` | `owner/repo` 简写糖 → 归一化到 `github.com` |
| `git-subdir` | object | `url`, `path`, `ref?`, `sha?` | Git 仓库子目录作为 plugin（monorepo，sparse clone）|
| `cnb` | object | `repo`, `ref?`, `sha?` | `group/project` 简写糖 → 归一化到 `cnb.cool`（国内主流 Git 平台）|

### 5.2 各类型字段表

#### 5.2.1 `github`

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `repo` | string | 是 | GitHub 仓库 `owner/repo` 格式 |
| `ref` | string | 否 | 分支或 tag（默认仓库默认分支） |
| `sha` | string | 否 | 完整 40 字符 commit SHA |

#### 5.2.2 `url`

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `url` | string | 是 | 完整 Git URL（`https://` 或 `git@`）；`.git` 后缀可选 |
| `ref` | string | 否 | 分支或 tag |
| `sha` | string | 否 | 完整 40 字符 commit SHA |

#### 5.2.3 `git-subdir`

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `url` | string | 是 | Git URL；亦接受 GitHub 简写 `owner/repo` 或 SSH URL |
| `path` | string | 是 | 子目录路径；**必须指向 plugin 根**（即被 marketplace 条目作为 plugin 引用的目录），**不可指向另一个 marketplace 根**（含 `marketplace.json`）。是否要求 `plugin.json` 存在见 §6 |
| `ref` | string | 否 | 分支或 tag |
| `sha` | string | 否 | 完整 40 字符 commit SHA |

#### 5.2.4 `cnb`

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `repo` | string | 是 | CNB 仓库 `group/project` 格式 |
| `ref` | string | 否 | 分支或 tag |
| `sha` | string | 否 | 完整 40 字符 commit SHA |

> Claude Code 客户端不识别 `cnb` 类型；TFRobotServer 引入是为了国内 Git 生态。归一化逻辑与 `github` 完全同构 —— D3 Loader 将 `{source: "cnb", repo: "g/p"}` 转换为 `https://cnb.cool/g/p.git` 走 git clone。

**注**：Portal 直接上传 (Upload) 与在线编辑 (Inline-Edit) **不属于 marketplace.json 范围** —— 它们是平台内部 Plugin/SKILL 分发渠道，绕过 marketplace catalog 直接进入 registry，不在本规范覆盖。

### 5.3 Marketplace source vs Plugin source（两套独立 schema）

Marketplace 体系实际上由 **两套互不交集的 source schema** 组成。它们通过出现位置区分，靠 `source.source` 字段的 union 类型判定，**不是嵌套递归**。

| 维度 | **Marketplace source** | **Plugin source** |
| --- | --- | --- |
| 控制什么 | 从哪里拉 `marketplace.json` catalog（整个目录）| 从哪里拉单个 plugin |
| 出现位置 | 用户在 Portal 添加 marketplace 时填写 | `marketplace.json` 中每个 plugin 条目的 `source` 字段 |
| TFRobotServer v1 实施类型 | `url` / `github` / `git` / `cnb`（4 类） | 相对路径 / `url` / `github` / `git-subdir` / `cnb`（5 类）|
| 是否支持 `sha` 锁版本 | 仅 `ref` 分支/tag，**不**支持 `sha` | 同时支持 `ref` 与 `sha` 精确锁定 commit |

> 两侧都仅接受 Git 引用，`github`/`cnb` 都是简写糖到不同 host 的 `git` —— 命名差异只为 UX 与 schema 校验，底层加载链路一致。

**关键交集为空（disjoint union）**：

| 类型 | 仅在 Marketplace source | 仅在 Plugin source |
| --- | --- | --- |
| `git` | ✅ —— marketplace 用 `git` 引用任意 Git URL | ❌ —— plugin 同语义用 `url`，discriminator 不同 |
| `git-subdir` | ❌ | ✅ —— marketplace **不能**用 git-subdir |
| 相对路径 `"./xxx"` | ❌ | ✅ —— 引用 marketplace 仓库内的 plugin 目录 |

> `cnb` 在 Marketplace source 与 Plugin source **两侧都存在**：marketplace catalog 与 plugin 都可托管于 CNB。两侧 `cnb` 字段定义独立（marketplace 侧多 `path`/`sparsePaths`）。

实际后果：

* `marketplace.json` 里的 plugin 条目**永远指向"plugin 源"**，从不指向"另一个 marketplace"
* 即便某 plugin 源恰好是另一个仓库（该仓库可能也对外提供 marketplace 接口），那个仓库的 marketplace 身份**不会被这条 plugin 条目触发** —— 装载器只会去 `git-subdir.path` 子目录读 `plugin.json`，不会去仓库根读 `marketplace.json`
* 详细装载流程见 [D-loading-behavior.md](loading-behavior.md) §"git-subdir 加载链路"

## 6. plugin.json 字段规范

> Plugin manifest，路径 `<plugin>/.tfrobot-plugin/plugin.json`。字段设计参考 [Claude Code Plugin Reference](https://code.claude.com/docs/en/plugins)。
>
> **条件必需**：是否存在 `plugin.json` 由 marketplace 条目的 `strict` 字段决定（§4.4）。装载行为细节、缺失兜底、冲突检测见 [D-loading-behavior.md](loading-behavior.md)。

### 6.1 必填字段

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `name` | string | Plugin 标识；同时作为 SKILL 命名空间前缀（如 `/<plugin>:<skill>`） |

### 6.2 可选字段

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `description` | string | Plugin 描述 |
| `version` | string | 版本。设置后用户仅在变更时收到更新；省略则用 Git commit SHA |
| `author` | object | `{name: string (必需), email?: string}` |
| `homepage` | string | 主页 URL |
| `repository` | string | 源码仓库 URL |
| `license` | string | SPDX license |
| `keywords` | array&lt;string&gt; | 关键字 |

### 6.3 组件目录（plugin 根，非 `.tfrobot-plugin/` 内）

| 目录 / 文件 | 用途 |
| --- | --- |
| `skills/` | SKILL 包（每个 `<name>/SKILL.md`）—— 本仓库消费的核心 |
| `commands/` | 平铺 `.md` 命令文件（TFRobotServer 不消费） |
| `agents/` | Agent 定义（TFRobotServer 不消费） |
| `hooks/hooks.json` | Hook 配置（TFRobotServer 不消费） |
| `.mcp.json` | MCP server 配置（TFRobotServer 不消费） |
| `.lsp.json` | LSP server 配置（TFRobotServer 不消费） |
| `monitors/monitors.json` | 后台 monitor 配置（TFRobotServer 不消费） |
| `bin/` | Bash PATH 可执行（TFRobotServer 不消费） |
| `settings.json` | Plugin 默认 settings（TFRobotServer 不消费） |

> **TFRobotServer v1 消费**：仅 `skills/` 子树。Plugin 内其他 Claude Code 私有组件 **识别但忽略不报错**，保证仓库可在两端互操作。

### 6.4 plugin.json 完整示例

```json
{
  "name": "data-toolkit",
  "description": "Data ingestion / cleansing / report SKILLs",
  "version": "1.2.0",
  "author": {
    "name": "TFRobotServer Team",
    "email": "tfs@turingfocus.com"
  },
  "homepage": "https://docs.turingfocus.com/skills/data-toolkit",
  "repository": "https://github.com/turingfocus/tfs-data-toolkit",
  "license": "MIT",
  "keywords": ["data", "etl", "pandas"]
}
```

## 7. SKILL 层（指向 A5）

SKILL 目录 `<plugin>/skills/<skill-name>/` 内的所有内容契约由 [A5 协议规范](../skill/protocol-v1.md) 定义，本规范不重述。要点提示：

* SKILL.md（必需）：A5 §3 frontmatter + A5 §4 body
* `.skillenv`（可选）：A5 §5
* `.cubesandbox.dockerfile` / `.e2b.dockerfile`（条件必需）：A5 §6.3
* 标准目录 `scripts/` / `references/` / `assets/`：A5 §2
* 占位符 `TFROBOT_SKILL_DIR` / `TFROBOT_SESSION_ID` / `TFROBOT_ROBOT_ID`：A5 §7
* 内置 `skills` 工具：A5 §8

## 8. Curator / Aggregator 模式

一个 marketplace 可以通过 `git-subdir` 引用**其他仓库中的 plugin**，即使该仓库自身也是一个 marketplace —— 这是合法的 curator 模式。

**前提**：`git-subdir.path` 必须落在 plugin 边界，不能指向 marketplace 边界（含 `marketplace.json`）。`plugin.json` 是否需要存在由 marketplace 条目 `strict` 字段决定，详见 §6 与 [D-loading-behavior.md](loading-behavior.md)。

```
turingfocus-curated/                                ← curator marketplace
  .tfrobot-plugin/marketplace.json
      └── plugins: [
              {
                "name": "data-toolkit",
                "source": {
                  "source": "git-subdir",
                  "url": "https://github.com/vendor-x/skills.git",
                  "path": "plugins/data/data-toolkit"
                                                    ← 指向 plugin 边界
                }
              }
          ]

vendor-x/skills/                                    ← provider marketplace（独立 catalog）
  .tfrobot-plugin/marketplace.json                  ← vendor-x 自营消费者走这里
  plugins/data/data-toolkit/                        ← plugin 实体（curator 引用这里）
    .tfrobot-plugin/plugin.json                     ← 条件必需，见 §6
    skills/...
```

**物理隔离保证**：marketplace 与 plugin 的 manifest 文件名不同（`marketplace.json` vs `plugin.json`），路径错位时装载即失败；即便某 plugin 目录在 `strict: false` 模式下不带 `plugin.json`，也不会被误识别为 marketplace（不含 `marketplace.json`）。

**Schema 隔离保证**：marketplace.json 里的 plugin 条目走 **Plugin source schema**（§5.4），永远只指向 plugin 源；即便 plugin 源恰好是另一个 repo（碰巧也带 marketplace 接口），那个 marketplace 身份**不会被触发**。两套 source schema 不交集是规范层面的根因，物理路径错位失败只是兜底。

**双重身份 repo 的加载独立性**：一个 repo 可同时承担两种身份 —— 根目录是自己的 marketplace（让粉丝直接订阅），子目录被第三方 marketplace 用 `git-subdir` 引用（让更大目录下的用户能发现）。这两条加载路径**不共享 clone**：

| 视角 | 装载器行为 |
| --- | --- |
| 用户加 `vendor-x/skills` 为 marketplace | 整个 repo clone 到 marketplace cache，读根 `marketplace.json`，列出 vendor-x 全套 plugin |
| 用户从 `turingfocus-curated` 装 `data-toolkit` | 独立 sparse clone `vendor-x/skills`，只提子目录，作为单个 plugin 安装；**不读** vendor-x repo 根的 `marketplace.json` |

代价是磁盘上同一 repo 可能被独立拉取两次；收益是两条路径完全解耦、cache 模型简单（避免一个 cache 既要支持 marketplace 视图又要支持 plugin 视图）。详细流程见 [D-loading-behavior.md](loading-behavior.md)。

**设计带来的好处**：

1. **plugin 作者一个 repo 多用**：根目录自营 marketplace + 子目录被第三方 curator 引用，两种身份并行不悖
2. **Curator marketplace 不必托管代码**：仅在 `marketplace.json` 里写指针（含 `sha` 锁定版本），实际内容仍归属各 plugin 作者
3. **粒度精确**：可以从一个 monorepo 挑某个子目录作为 plugin，不强迫作者拆 repo
4. **`sha` 锁版本**：curator 可以"快照" plugin 某个 commit，作者后续改动不会立即影响 curator 用户

**消费 Claude Code 标准仓库的可能性**：因 manifest 路径独立（`.tfrobot-plugin/` vs `.claude-plugin/`），TFRobotServer 平台默认不消费仅含 `.claude-plugin/` manifest 的仓库；若需互操作，由 Module D 实施时决定是否支持"路径回退识别"（v1 不收入，见 §10）。

## 9. 与 Claude Code Marketplace 关系矩阵

| 维度 | 关系 |
| --- | --- |
| `marketplace.json` 顶层字段 | 字段名/语义沿用 Claude Code 设计 |
| `marketplace.json` 必填字段 | 字段名/语义沿用 |
| Plugin 条目字段 | 字段名/语义沿用 |
| `plugin.json` 字段 | 字段名/语义沿用 |
| Source 类型枚举（`github` / `url` / `git-subdir` / `npm` / 相对路径） | 字段名/语义沿用 |
| Source 类型扩展（`cnb`） | TFRobotServer 引入用于国内 Git 生态；Claude Code 客户端不识别即跳过 |
| **Manifest 路径** | **独立**：`.tfrobot-plugin/` 而非 `.claude-plugin/`；同一份仓库**不能**被两端直接互换消费 |
| **Marketplace 保留名空间** | **独立**：`tfrobot-` / `turingfocus-` / `tfs-` 前缀；与 Claude Code 保留名空间互不影响 |
| Plugin 内组件（`commands/` / `agents/` / `hooks/` / `mcpServers` / `lspServers` / `monitors/` / `bin/` / `settings.json`） | TFRobotServer 不消费但容忍存在 |
| SKILL 层私有内容（`.cubesandbox.dockerfile` / `.e2b.dockerfile` / `.skillenv` / 占位符 / 内置工具） | 由 A5 协议定义；不属本规范范围 |
| 双轨发布可行性 | 作者可同时维护 `.tfrobot-plugin/` 与 `.claude-plugin/` 两套 manifest，分别面向两端发布 |

## 10. v1 不引入（与理由）

| 不引入项 | 理由 |
| --- | --- |
| 平台托管 Git 仓库 / SaaS 化 Marketplace | v1 仅作 Git pull 消费方；托管侧由 Portal / CNB 等承担 |
| 完整性签名校验（GPG / Sigstore） | Epic 范围外；v1 信任 Git 提交历史 |
| Plugin 间依赖解析 / 版本约束 / `allowCrossMarketplaceDependenciesOn` 校验 | 字段可识别透传，但 v1 不强制约束 |
| 跨租户公共 Marketplace、CDN 加速 | Epic 范围外 |
| `npm` source 类型（marketplace 与 plugin 两侧）| 需要 node 运行时；TFRobotServer 是 Python 服务端，与栈不匹配 |
| `pip` source 类型（plugin 侧）| SaaS 服务**不允许任意 `pip install`**（安全约束）；plugin 内容须托管在可审计的 Git 仓库 |
| `file` / `directory` source 类型 | 多 Pod K8s 部署无共享本地 FS |
| `hostPattern` / `pathPattern` source 类型 | Claude Code 客户端 `strictKnownMarketplaces` 管控字段，服务端无对应入口 |
| `settings` source 类型 | 内联到 Claude Code 客户端 settings.json，与客户端 settings 强耦合 |
| 把 Portal Upload / Inline-Edit 纳入 marketplace.json | 两者属于平台内部 Plugin/SKILL 注入渠道，不经过 marketplace catalog |
| `.claude-plugin/` 路径回退识别 | v1 仅识别 `.tfrobot-plugin/`；不消费仅含 `.claude-plugin/` manifest 的 Claude Code 标准仓库；如需互操作由后续 ticket 评估 |
| Plugin 内 Claude Code 私有组件（`commands/` / `hooks/` / `agents/` / `mcpServers` / `lspServers` / `monitors/` / `bin/` / `settings.json`）| TFRobotServer 不消费；识别但忽略不报错 |

## 11. 准出对照（D1 子任务）

* [x] Marketplace 层级结构定义（§2）
* [x] `marketplace.json` JSON Schema —— 必填/可选字段表（§3）
* [x] Plugin 条目字段表（§4）+ strict mode 语义（§4.4）
* [x] Plugin source 类型枚举 + 各类型字段表（§5）
* [x] `plugin.json` JSON Schema 字段表（§6）
* [x] SKILL 层引用 A5（§7）
* [x] Curator / Aggregator 模式合法性与边界（§8）
* [x] 与 Claude Code Marketplace 兼容性矩阵（§9）
* [x] v1 不引入项（§10）

依赖下游子任务（实施层，本规范不细化）：

* D2 `marketplace.json` / `plugin.json` Pydantic schema 落地 + Postgres registry schema
* D3 多 source 类型的拉取实施（含 `github` / `cnb` 简写糖归一化）
* D4 双层状态 Reconciler 与 MinIO 物化
* D5 Marketplace/Plugin/SKILL 三态生命周期与 Robot/Factory 协作
* D6 端到端 Marketplace pull 跑通 + 示例 marketplace 仓库

## 12. 参考

* [Claude Code Plugin Marketplaces 官方规范](https://code.claude.com/docs/en/plugin-marketplaces)
* [Claude Code Plugins 官方规范](https://code.claude.com/docs/en/plugins)
* [Anthropic 官方 marketplace catalog（claude-plugins-official）](https://github.com/anthropics/claude-code)
* [Agent Skills 开放标准](https://agentskills.io/specification)

## 附录 A：与 A5 协议规范的边界

```
┌────────────────────────────────────────────────────────────────────────┐
│ SKILL 文件夹内容契约                ◄── A5 协议规范                     │
│ (frontmatter / dirs / runtime /                                        │
│  placeholders / skills tool / ...)                                     │
│           │                                                            │
│           │ 多种分发渠道                                                │
│           ▼                                                            │
│ ┌────────────┐  ┌────────────┐  ┌──────────────────────────────────┐  │
│ │ Portal     │  │ Portal     │  │ Marketplace                      │  │
│ │ 直接上传    │  │ 在线编辑    │  │ <repo>/.tfrobot-plugin/           │  │
│ │ (非本规范) │  │ (非本规范)  │  │   marketplace.json               │  │
│ │            │  │            │  │ + plugins/<x>/.tfrobot-plugin/    │  │
│ │            │  │            │  │   plugin.json                    │  │
│ │            │  │            │  │ + plugins/<x>/skills/<y>/        │  │
│ └────────────┘  └────────────┘  └──────────────────────────────────┘  │
│                                       ▲                                │
│                                       └── 本规范                       │
│                                                                        │
│                       共同物化目标                                      │
│                       ┌───────────────────┐                            │
│                       │ MinIO + Postgres  │   ◄── A5 §10 / 实施层      │
│                       │ registry          │                            │
│                       └───────────────────┘                            │
└────────────────────────────────────────────────────────────────────────┘
```

A5 管"SKILL 内容契约"；本规范管"Marketplace / Plugin 仓库结构与 JSON Schema"。
