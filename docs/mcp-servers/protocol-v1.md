# MCP Server 配置规范 v1

> 范围：TFRobot Plugin 内 `mcp-servers/` 子树的目录约定与配置 schema 摘要。
> 读者：Plugin 作者（撰写指南）+ 接入方实施工程师（加载使用层最小契约）。
> Schema 权威：[A2C-SMCP 协议规范 § MCP Server 配置结构](https://doc.turingfocus.cn/a2c-smcp/latest/specification/data-structures/#mcp-server-配置结构)（当前对齐版本 **v0.2.0**）。

## 0. 总览

### 0.1 这是什么

TFRobot Plugin 可在 `<plugin>/mcp-servers/` 下声明若干 MCP Server 配置；每个配置是一份独立 JSON 文件，结构遵守 A2C-SMCP `MCPServerConfig` 联合类型。Robot 在加载 Plugin 时把这些配置交给 Computer 侧，由 Computer 通过 A2C-SMCP 暴露给 LLM 工具循环。

### 0.2 协议边界

| 维度 | 本规范 | A2C-SMCP |
| --- | --- | --- |
| Plugin 内 `mcp-servers/` 目录约定 | ✅ 规定 | 不涉及 |
| 单份 MCP Server 配置 JSON 结构 | ❌ 摘要 / 速查 | ✅ 权威 |
| 占位符 `${input:<id>}` 语法 | ❌ 摘要 / 速查 | ✅ 权威 |
| MCP 连接生命周期、协议握手、tool 调用语义 | 不涉及 | ✅ 权威 |

**v1 不引入**：MCP 嵌入 `plugin.json`（保持文件式而非内嵌）、跨 plugin 共享 `inputs.json`（v1 限 plugin 范围）、Schema 拓展（v1 完全等同 A2C-SMCP v0.2.0）。

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
| 文件名 ↔ 配置内 `name` | **必须严格相等**（去掉 `.json` 后缀） |
| `inputs.json` 作用域 | Plugin 范围内共享；该 plugin 下所有 server.json 均可引用其中的 input |

### 1.2 加载方义务

Robot / 加载方装载 plugin 时：

1. 若 `mcp-servers/` 存在且非空 → 枚举所有 `*.json`（除 `inputs.json`）
2. 对每份 JSON 按 §2/§3 解析为 `MCPServerConfig`
3. 收集 `inputs.json`（若存在）；按 A2C-SMCP 约定的 `${input:<id>}` 语法在传输参数 / 环境变量 / headers 等字段内做占位符展开
4. 转交 Computer 侧；执行后端 / 凭证 / 进程管理由 Computer 按 A2C-SMCP 自行处理

## 2. 通用字段（BaseMCPServerConfig）

| 字段 | 类型 | 必填 | 含义 |
| --- | --- | --- | --- |
| `name` | string | 是 | MCP Server 名称；必须等于文件名（去掉 `.json`） |
| `type` | enum | 是 | 传输模式：`stdio` / `streamable` / `sse` |
| `disabled` | boolean | 是 | 是否禁用；加载时跳过 |
| `forbidden_tools` | array&lt;string&gt; | 是 | 工具名黑名单 |
| `tool_meta` | object | 是 | 工具名 → ToolMeta 映射（见 §5） |
| `default_tool_meta` | ToolMeta \| null | 否 | 默认 ToolMeta；具体工具未配置时使用 |
| `vrl` | string \| null | 否 | VRL 脚本，用于动态转换工具返回值 |
| `server_parameters` | object | 是 | 传输参数；类型由 `type` 字段分派（§3） |

> `type = "streamable"` 对应 MCP 官方的 **Streamable HTTP** 传输模式。

## 3. 传输参数（按 `type` 分派）

### 3.1 type = "stdio"（MCPServerStdioParameters）

| 字段 | 类型 | 必填 | 含义 |
| --- | --- | --- | --- |
| `command` | string | 是 | 启动命令 |
| `args` | array&lt;string&gt; | 是 | 命令行参数 |
| `env` | object \| null | 是 | 环境变量 map |
| `cwd` | string \| null | 是 | 工作目录 |
| `encoding` | string | 是 | 文本编码，默认 `utf-8` |
| `encoding_error_handler` | enum | 是 | `strict` / `ignore` / `replace` |

### 3.2 type = "streamable"（MCPServerStreamableHttpParameters）

| 字段 | 类型 | 必填 | 含义 |
| --- | --- | --- | --- |
| `url` | string | 是 | 端点 URL |
| `headers` | object \| null | 是 | HTTP 请求头 |
| `timeout` | string | 是 | HTTP 超时（ISO 8601 duration，如 `"PT30S"`） |
| `sse_read_timeout` | string | 是 | SSE 读取超时（ISO 8601 duration） |
| `terminate_on_close` | boolean | 是 | 关闭时是否终止客户端会话 |

### 3.3 type = "sse"（MCPSSEParameters）

| 字段 | 类型 | 必填 | 含义 |
| --- | --- | --- | --- |
| `url` | string | 是 | 端点 URL |
| `headers` | object \| null | 是 | HTTP 请求头 |
| `timeout` | number | 是 | HTTP 超时（秒） |
| `sse_read_timeout` | number | 是 | SSE 读取超时（秒） |

## 4. 占位符输入（`inputs.json`）

`inputs.json` 是数组；每个元素是一个 input 定义，被 MCP Server 配置中的占位符（A2C-SMCP 约定的 `${input:<id>}` 语法）引用。

### 4.1 通用基础字段（MCPServerInputBase）

| 字段 | 类型 | 必填 | 含义 |
| --- | --- | --- | --- |
| `id` | string | 是 | 输入 ID（占位符引用键） |
| `description` | string | 是 | 描述 |
| `type` | enum | 是 | `promptString` / `pickString` / `command` |

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
| `options` | array&lt;string&gt; | 是 | 可选项列表 |
| `default` | string \| null | 否 | 默认值 |

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
| `alias` | string \| null | 否 | 工具别名，用于解决跨 server 同名冲突 |
| `tags` | array&lt;string&gt; \| null | 否 | 工具标签，用于分类 |

## 6. 完整示例

### 6.1 stdio 模式

`plugins/data-toolkit/mcp-servers/postgres-explorer.json`：

```json
{
  "name": "postgres-explorer",
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
| 当前对齐版本 | A2C-SMCP **v0.2.0** |
| 同步策略 | A2C-SMCP 升级到 v0.3+ 时，本规范摘要随其变更同步；以 A2C-SMCP 为准 |
| 不重复设计 | 本规范**不**为 MCP 配置发明替代字段、替代命名、替代占位符语法；歧义以 A2C-SMCP 为权威 |

## 8. v1 不引入

| 不引入项 | 理由 |
| --- | --- |
| MCP 配置内嵌 `plugin.json` | 文件式与 SKILL 子树设计对齐；JSON 嵌套膨胀风险 |
| 跨 plugin 共享 `inputs.json` | v1 限 plugin 范围；跨 plugin 共享需 registry 层支持，推迟到未来 |
| Schema 扩展（自定义传输类型、自定义占位符语法） | 完全对齐 A2C-SMCP 是 v1 的核心原则 |
| MCP 连接生命周期 / tool 调用语义 | 属于 A2C-SMCP 与 MCP 协议本身，不在本规范范围 |
