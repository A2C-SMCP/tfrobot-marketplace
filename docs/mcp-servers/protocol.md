# MCP Server 配置规范

> 范围：TFRobot Plugin 内 `mcp-servers/` 子树的目录约定与配置 schema 摘要。
> 读者：Plugin 作者（撰写指南）+ 接入方实施工程师（加载使用层最小契约）。
> Schema 权威：[A2C-SMCP 协议规范 § MCP Server 配置结构](https://doc.turingfocus.cn/a2c-smcp/latest/specification/data-structures/#mcp-server-配置结构)（当前对齐版本 **v0.3.2**）。

## 0. 总览

### 0.1 这是什么

TFRobot Plugin 可在 `<plugin>/mcp-servers/` 下声明若干 MCP Server 配置；每个配置是一份独立 JSON 文件，结构遵守 A2C-SMCP `MCPServerConfig` 联合类型。Robot 在加载 Plugin 时把这些配置交给 Computer 侧，由 Computer 通过 A2C-SMCP 暴露给 LLM 工具循环。

### 0.2 协议边界

| 维度 | 本规范 | A2C-SMCP |
| --- | --- | --- |
| Plugin 内 `mcp-servers/` 目录约定 | ✅ 规定 | 不涉及 |
| 单份 MCP Server 配置 JSON 结构 | ❌ 摘要 / 速查 | ✅ 权威 |
| 占位符语法（`${input:<id>}` / `${env:<name>}` / 预定义变量） | ❌ 摘要 / 速查 | ✅ 权威 |
| MCP 连接生命周期、协议握手、tool 调用语义 | 不涉及 | ✅ 权威 |

**本规范不引入**：MCP 嵌入 `plugin.json`（保持文件式而非内嵌）、跨 plugin 共享 `inputs.json`（限 plugin 范围）、Schema 拓展（完全等同 A2C-SMCP v0.3.2）。

## 1. 目录约定

```
<plugin>/
  .tfrobot-plugin/plugin.json
  mcp-servers/                      # 可选 —— 若存在，至少含 1 个 server.json
    <server-name>.json              # 单个 MCP Server 配置（§2 / §3）
    inputs.json                     # 可选 —— 占位符输入定义（§4）
```

| 路径 | 必填 | 内容 |
| --- | --- | --- |
| `<plugin>/mcp-servers/` | 否；存在则非空 | MCP Server 配置子树 |
| `<plugin>/mcp-servers/<server-name>.json` | 子树存在时至少 1 个 | 单个 MCP Server 配置 |
| `<plugin>/mcp-servers/inputs.json` | 否 | 占位符输入定义数组 |

### 1.1 命名与对齐约束

| 项 | 约束 |
| --- | --- |
| `<server-name>` 字符集 | `[a-z0-9-]`，kebab-case，不以 `-` 开头/结尾，无连续 `--`（与 SKILL 名一致） |
| 文件名 ↔ 配置内 `name` | **必须严格相等**（去掉 `.json` 后缀）。注意 `name` 是纯 display、非身份（唯一身份是 `bundle_id`，见 §2） |
| `inputs.json` 作用域 | Plugin 范围内共享；该 plugin 下所有 server.json 均可引用其中的 input。入池时 id 前缀化为 `<plugin>@<marketplace>/<id>`（跨 plugin 消歧）；server.json 内写**裸** `${input:<id>}`，解析时经前缀回退命中 |

### 1.2 加载方义务

Robot / 加载方装载 plugin 时：

1. 若 `mcp-servers/` 存在且非空 → 枚举所有 `*.json`（除 `inputs.json`）
2. 对每份 JSON 按 §2/§3 解析为 `MCPServerConfig`
3. 收集 `inputs.json`（若存在）；按 A2C-SMCP 约定的占位符语法（`${input:<id>}` / `${env:<name>}` / 预定义 `${userHome}` `${pathSeparator}`）在传输参数 / 环境变量 / headers 等字段内做占位符展开
4. 转交 Computer 侧；执行后端 / 凭证 / 进程管理由 Computer 按 A2C-SMCP 自行处理

## 2. 通用字段（BaseMCPServerConfig）

| 字段 | 类型 | 必填 | 含义 |
| --- | --- | --- | --- |
| `name` | string | 是 | MCP Server 名称（**纯 display**，人类可读）。必须等于文件名（去掉 `.json`）；允许碰撞、**永不做键/寻址**——唯一身份是 `bundle_id` |
| `bundle_id` | string \| null | 否 | MCP Server **唯一身份**（软件级 BundleID，0.3.0 起）。省略时由 `name` 经确定性算法派生（解析后恒有值、不回写配置源）；显式值须为 `[A-Za-z0-9_-]`、**禁 `.` 与连续 `__`**；同一 `bundle_id` 视为同一软件，不允许多开 |
| `type` | enum | 是 | 传输模式：`stdio` / `streamable` / `sse` |
| `disabled` | boolean | 否 | 是否禁用；加载时跳过（默认 `false`） |
| `forbidden_tools` | array&lt;string&gt; | 否 | 工具名黑名单（默认空数组） |
| `tool_meta` | object | 否 | 工具名 → ToolMeta 映射（默认空对象；见 §5） |
| `default_tool_meta` | ToolMeta \| null | 否 | 默认 ToolMeta；具体工具未配置时使用 |
| `vrl` | string \| null | 否 | VRL 脚本，用于动态转换工具返回值 |
| `envFile` | string \| null | 否 | **SDK 加性字段（协议未收录）**：VS Code 对标（v0.2.1 #65），stdio 专属；spawn 时从 `.env` 加载 KEY=VALUE 进 stdio server 的 env，显式 `env` 同名项覆盖 `envFile`；非 stdio（sse / http 无 env）忽略 |
| `oauth` | object \| null | 否 | **SDK 加性字段（协议未收录）**：`streamable` 专属 OAuth 配置（对齐 Rust `HttpServerConfig.oauth`） |
| `server_parameters` | object | 是 | 传输参数；类型由 `type` 字段分派（§3） |

> `type = "streamable"` 对应 MCP 官方的 **Streamable HTTP** 传输模式。

!!! note "BundleID 模型：name 非身份，bundle_id 才是（0.3.0 起）"

    文件名 = `name` 的目录约定**未变**（文件 stem 仍强制等于 `name`），但 `name` 已降级为纯 display：允许碰撞、永不做键/寻址。MCP Server 的唯一身份是 **`bundle_id`**——`client:get_config` 的 `servers` 字典 key、SKILL mcp 形态名的 `<server>` 段、错误码 `meta.mcp_server` 全用 `bundle_id`。聚合后暴露给 LLM 的工具名为 `{bundle_id}__{alias ?? 原始工具名}`（alias 语义见 §5）。

    **缺省派生（`bundle_id` 省略时）**：按 Unicode 码点规范化 `name`——非 `[A-Za-z0-9_-]` 码点替换为 `_`、折叠连续 `_`、裁剪首尾 `[_-]`；结果非空即 `bundle_id`；为空（全符号 / CJK / 空串）则 `bundle_` + SHA-256(connection-identity 字节串)[:8] 小写 hex。算法逐字节确定、跨 SDK 一致（协议仓托管[一致性测试向量](https://doc.turingfocus.cn/a2c-smcp/latest/specification/data-structures/#bundleid-conformance)），在加载/注册期完成（derive-on-load），**不回写配置源**（如 `<server-name>.json`）。

    显式 `bundle_id` 含 `.` / `__` / 字符集越界属**非法配置**——按平台诊断处理（rust 侧在 parse 期整个 plugin 判废）；省略 `bundle_id` 不算错误（触发缺省生成）。注意规范化非单射：`my server` / `my-server` 均 → `my_server`，两个都叫 `everything` 的 server 缺省生成后会撞同一 `bundle_id` → 多开冲突，由配置人员显式指定 `bundle_id` 消歧。

## 3. 传输参数（按 `type` 分派）

### 3.1 type = "stdio"（MCPServerStdioParameters）

| 字段 | 类型 | 必填 | 含义 |
| --- | --- | --- | --- |
| `command` | string | 是 | 启动命令 |
| `args` | array&lt;string&gt; | 是 | 命令行参数 |
| `env` | object \| null | 是 | 环境变量 map |
| `cwd` | string \| null | 是 | 工作目录 |
| `encoding` | string | 是 | 文本编码，默认 `utf-8`（协议 TypedDict 必填；⚠️ 双 SDK 当前均未消费，serde 静默忽略） |
| `encoding_error_handler` | enum | 是 | `strict` / `ignore` / `replace`（同上：协议必填、SDK 暂未消费） |

### 3.2 type = "streamable"（MCPServerStreamableHttpParameters）

| 字段 | 类型 | 必填 | 含义 |
| --- | --- | --- | --- |
| `url` | string | 是 | 端点 URL |
| `headers` | object \| null | 是 | HTTP 请求头 |
| `timeout` | string | 是 | HTTP 超时（ISO 8601 duration，如 `"PT30S"`） |
| `sse_read_timeout` | string | 是 | SSE 读取超时（ISO 8601 duration） |
| `terminate_on_close` | boolean | 是 | 关闭时是否终止客户端会话 |
| `oauth` | object \| null | 否 | **SDK 加性字段（协议未收录）**：OAuth 配置（对齐 Rust `HttpServerConfig.oauth`），`streamable` 专属 |

### 3.3 type = "sse"（MCPSSEParameters）

| 字段 | 类型 | 必填 | 含义 |
| --- | --- | --- | --- |
| `url` | string | 是 | 端点 URL |
| `headers` | object \| null | 是 | HTTP 请求头 |
| `timeout` | number | 是 | HTTP 超时（秒） |
| `sse_read_timeout` | number | 是 | SSE 读取超时（秒） |

## 4. 占位符输入（`inputs.json`）

`inputs.json` 是数组；每个元素是一个 input 定义，被 MCP Server 配置中的占位符引用。

**占位符语法面**（A2C-SMCP 权威；SDK 实际支持）：

| 占位符 | 语义 |
| --- | --- |
| `${input:<id>}` | 经 input 解析链取值（本节定义的 input 的注入值） |
| `${env:<NAME>}` | 进程环境变量；缺失 → 空串 + WARN |
| `${userHome}` / `${pathSeparator}` | 预定义变量（用户主目录 / 路径分隔符） |
| `${workspaceFolder}` | ⚠️ 已弃用（随 workdir 概念瘦身移除，仅 WARN；改用 `${input:<id>}` 或绝对路径） |

未知占位符保持原样（向后兼容）。

### 4.1 通用基础字段（MCPServerInputBase）

| 字段 | 类型 | 必填 | 含义 |
| --- | --- | --- | --- |
| `id` | string | 是 | 输入 ID（占位符引用键） |
| `description` | string | 是 | 描述 |
| `type` | enum | 是 | `promptString` / `pickString` / `command`（协议 Literal **小写**。⚠️ rust-sdk 当前仅接受 PascalCase（`PromptString`），按本文档示例写的配置在 Rust 侧会被判废——已知 Rust 缺口，应以协议为准，建议 Rust 补 alias） |

### 4.2 type = "promptString"

附加字段：

| 字段 | 类型 | 必填 | 含义 |
| --- | --- | --- | --- |
| `default` | string \| null | 否 | 默认值 |
| `password` | boolean \| null | 否 | 是否为密码（隐藏输入） |

### 4.3 type = "pickString"

附加字段：

| 字段 | 类型 | 必填 | 含义 |
| --- | --- | --- | --- |
| `options` | array&lt;PickStringOption&gt; | 是 | 可选项列表（结构化 `{label, value}`） |
| `default` | string \| null | 否 | 默认值（若存在且非 None，MUST 匹配至少一个 `option.value`） |

`PickStringOption`：`label`（展示标签，UI 呈现）+ `value`（传值，注入 `${input:<id>}` 的实际值）——两者解耦，选项可以是机器值（区域码 / URL / ID）而不牺牲可读性。

约束（MUST）：

* `options` 至少 1 项；每个 `label` / `value` 非空字符串
* `label` 与 `value` 均允许重复（不要求唯一）；SDK / client MUST NOT 按 `value` 反推原选 `label`
* `default` 显式 `null` 视为无默认（不拒绝，解析时走无值解析链）

!!! warning "0.3.2 破坏性变更：旧 `options: list[str]` 直接拒绝"

    0.3.2 起 `options` 为结构化 `[{label, value}]` 列表。旧形式 `"options": ["a", "b"]`（字符串数组）是**非法 config**，SDK MUST 以 `validation` 错误拒绝（报错应指路新结构）。**不提供 alias、不设迁移期**——协议尚未正式上线，无存量兼容包袱。

    **双 SDK 采纳状态**：rust-sdk / python-sdk 的 `develop` 目前仍为 `Vec<String>`（protocol #48 半部推进中，python-sdk v0.3.2 实测亦为 `list[str]`）。文档既定策略是**以 A2C-SMCP 为权威**——作者应按新结构撰写；SDK 跟进后再完全互通。

### 4.4 type = "command"

附加字段：

| 字段 | 类型 | 必填 | 含义 |
| --- | --- | --- | --- |
| `command` | string | 是 | 要执行的命令 |
| `args` | object \| null | 否 | 命令参数 map |

## 5. ToolMeta

工具级元数据，用于 §2 中的 `tool_meta` 与 `default_tool_meta` 字段：

| 字段 | 类型 | 必填 | 含义 |
| --- | --- | --- | --- |
| `auto_apply` | boolean \| null | 否 | 是否自动应用（跳过二次确认） |
| `ret_object_mapper` | object \| null | 否 | 返回值对象映射，用于统一不同 MCP 工具的返回格式 |
| `alias` | string \| null | 否 | 工具别名。仅替换 exposed_tool_name 的**工具名部分**——最终暴露名恒为 `{bundle_id}__{alias ?? 原始工具名}`，`{bundle_id}__` 前缀不变（0.3.0 起跨 server 冲突已由 bundle_id 前缀结构性解决；alias 用于适配下游命名约束，如含连字符 / 保留字的工具名） |
| `tags` | array&lt;string&gt; \| null | 否 | 工具标签，用于分类 |

## 6. 完整示例

### 6.1 stdio 模式

`plugins/data-toolkit/mcp-servers/postgres-explorer.json`：

```json
{
  "name": "postgres-explorer",
  "bundle_id": "postgres-explorer",
  "type": "stdio",
  "disabled": false,
  "forbidden_tools": ["drop_table", "truncate"],
  "tool_meta": {
    "describe_table": {
      "auto_apply": true,
      "tags": ["read-only"]
    }
  },
  "default_tool_meta": null,
  "vrl": null,
  "server_parameters": {
    "command": "uvx",
    "args": ["mcp-server-postgres", "--host", "${input:db-host}"],
    "env": {
      "POSTGRES_PASSWORD": "${input:db-password}"
    },
    "cwd": null,
    "encoding": "utf-8",
    "encoding_error_handler": "strict"
  }
}
```

> `bundle_id` 可省略（缺省生成自 `name`，见 §2）；`disabled` / `forbidden_tools` / `tool_meta` / `default_tool_meta` / `vrl` 均有默认值，可省略。

`plugins/data-toolkit/mcp-servers/inputs.json`：

```json
[
  {
    "id": "db-host",
    "description": "PostgreSQL 主机地址",
    "type": "promptString",
    "default": "localhost"
  },
  {
    "id": "db-password",
    "description": "PostgreSQL 密码",
    "type": "promptString",
    "password": true
  }
]
```

### 6.2 streamable 模式

`<plugin>/mcp-servers/web-search.json`：

```json
{
  "name": "web-search",
  "type": "streamable",
  "disabled": false,
  "forbidden_tools": [],
  "tool_meta": {},
  "server_parameters": {
    "url": "https://mcp.example.com/web-search",
    "headers": {
      "Authorization": "Bearer ${input:api-token}"
    },
    "timeout": "PT30S",
    "sse_read_timeout": "PT5M",
    "terminate_on_close": true
  }
}
```

### 6.3 sse 模式

`<plugin>/mcp-servers/log-stream.json`：

```json
{
  "name": "log-stream",
  "type": "sse",
  "disabled": false,
  "forbidden_tools": [],
  "tool_meta": {},
  "server_parameters": {
    "url": "https://mcp.example.com/log-stream",
    "headers": null,
    "timeout": 30.0,
    "sse_read_timeout": 300.0
  }
}
```

## 7. 权威来源与版本对齐

| 项 | 值 |
| --- | --- |
| Schema 权威 | [A2C-SMCP 协议规范 § MCP Server 配置结构](https://doc.turingfocus.cn/a2c-smcp/latest/specification/data-structures/#mcp-server-配置结构) |
| 当前对齐版本 | A2C-SMCP **v0.3.2** |
| 同步策略 | 随 A2C-SMCP **协议版本**同步（以协议为权威）；SDK 补丁版本（bugfix 末位）不构成同步触发 |
| 不重复设计 | 本规范**不**为 MCP 配置发明替代字段、替代命名、替代占位符语法；歧义以 A2C-SMCP 为权威 |

## 8. 不引入

| 不引入项 | 理由 |
| --- | --- |
| MCP 配置内嵌 `plugin.json` | 文件式与 SKILL 子树设计对齐；JSON 嵌套膨胀风险 |
| 跨 plugin 共享 `inputs.json` | 限 plugin 范围；跨 plugin 共享需 registry 层支持，推迟到未来 |
| Schema 扩展（自定义传输类型、自定义占位符语法） | 完全对齐 A2C-SMCP 是核心原则 |
| MCP 连接生命周期 / tool 调用语义 | 属于 A2C-SMCP 与 MCP 协议本身，不在本规范范围 |
