# 配置 MCP Server

> 场景：给 Plugin 里的 SKILL 配上外部工具能力（数据库、搜索、自研服务……），让 LLM 工具循环能直接调用。
> Schema 权威是 [A2C-SMCP data-structures.md](https://doc.turingfocus.cn/a2c-smcp/latest/specification/data-structures/#mcp-server-配置结构)；本站 [MCP Server 配置规范](../mcp-servers/protocol.md) 只规定目录约定 + 字段速查。

## 第 1 步：选传输模式

| type | 适用 | 连接字段 |
| --- | --- | --- |
| `stdio` | 本地启动的命令行进程（uvx / node 等） | `command` + `args` + `env` |
| `streamable` | HTTP 端点（MCP 官方 Streamable HTTP） | `url` + `headers`（timeout 为 ISO 8601 duration） |
| `sse` | SSE 端点 | `url` + `headers`（timeout 为秒） |

## 第 2 步：写 `<server-name>.json`

放在 `<plugin>/mcp-servers/` 下，**文件名（去 `.json`）必须等于配置内 `name`**：

```json
{
  "name": "postgres-explorer",
  "bundle_id": "postgres-explorer",
  "type": "stdio",
  "disabled": false,
  "forbidden_tools": ["drop_table"],
  "tool_meta": {},
  "server_parameters": {
    "command": "uvx",
    "args": ["mcp-server-postgres", "--host", "${input:db-host}"],
    "env": { "POSTGRES_PASSWORD": "${input:db-password}" }
  }
}
```

关键约定：

- **`name` 是纯显示名**（允许碰撞、不做键/寻址）；唯一身份是 **`bundle_id`**——省略时由 `name` 确定性派生（不回写配置），显式值须 `[A-Za-z0-9_-]`、禁 `.` 与连续 `__`。
- 聚合后暴露给 LLM 的工具名 = `{bundle_id}__{alias ?? 原始工具名}`。
- `disabled` / `forbidden_tools` / `tool_meta` 均有默认值，可省略。

## 第 3 步：写 inputs.json（需要用户输入时）

同一 Plugin 下所有 server 共享 `mcp-servers/inputs.json`；server 配置里写**裸** `${input:<id>}`：

```json
[
  { "id": "db-host", "description": "数据库地址", "type": "promptString", "default": "localhost" },
  { "id": "db-password", "description": "数据库密码", "type": "promptString", "password": true },
  {
    "id": "region",
    "description": "部署区域",
    "type": "pickString",
    "options": [ { "label": "华东", "value": "cn-east" }, { "label": "华北", "value": "cn-north" } ]
  }
]
```

- input 类型：`promptString` / `pickString` / `command`（协议 Literal 小写）。
- `pickString` 的 `options` 是结构化 `[{label, value}]`——`label` 展示、`value` 注入，两者解耦。旧字符串数组形式是非法 config（0.3.2 破坏性变更）。
- 密码字段标 `password: true`（隐藏输入）。

## 第 4 步：占位符与校验

可用占位符全集：`${input:<id>}`、`${env:<NAME>}`、`${userHome}`、`${pathSeparator}`（`${workspaceFolder}` 已弃用）。未知占位符原样保留。

发布前用仓库校验脚本过一遍（镜像 A2C-SMCP SDK 校验边界）：`python3 -B scripts/validate_tfrobot_marketplace.py <仓库根>`。

## 参考

- [MCP Server 配置规范](../mcp-servers/protocol.md)（目录约定 + 字段速查）
- [A2C-SMCP § MCP Server 配置结构](https://doc.turingfocus.cn/a2c-smcp/latest/specification/data-structures/#mcp-server-配置结构)（schema 权威）
- [A2C-SMCP computer-mcp-config-guide](https://doc.turingfocus.cn/a2c-smcp/latest/guides/computer-mcp-config-guide/)（Computer 侧装配视角）
