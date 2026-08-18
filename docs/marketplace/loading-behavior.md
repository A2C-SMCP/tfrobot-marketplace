# Plugin manifest 加载行为参考（基于 Claude Code 实现观察）

> 关联：[protocol.md](protocol.md) 主规范
> 性质：**实施/运行时行为参考**，非规范强制约束。实施方建议复刻 Claude Code 这套行为以最大化兼容；SKILL/Plugin 作者用于理解装载时的取舍。
> 来源：[Claude Code 官方文档](https://code.claude.com/docs/en/plugin-marketplaces) + 公开源码观察（`pluginLoader.ts` / `schemas.ts`）+ **A2C 参考实现 SDK（python-sdk / rust-sdk）实测**。Claude Code 原版与双 SDK 有分歧处已显式标注（§1 / §6 / §7）——Module D 应**以双 SDK 实测为准**对齐 Computer 侧，避免行为分叉。

## 0. 为什么单列此文档

主规范 [protocol.md](protocol.md) 关心 **静态契约**（marketplace.json / plugin.json 的字段定义、路径约定、source 类型枚举）。

本文档关心 **动态行为与源码层观察**：

* `plugin.json` 缺失时装载是否成功？兜底字段从哪里来？
* `strict: false` 模式下两端冲突如何报错？
* 装载器是否"按约定自动发现"组件目录？
* Marketplace source 与 Plugin source 在源码 schema 层如何分离？
* `git-subdir` 的实际克隆流程是什么样？为什么"嵌套 marketplace"是错觉？
* 双重身份 repo 的两条加载路径是否共享 clone？

这些是参考实现的运行时语义，与规范契约同源但分层管理 —— 主规范的字段定义不应背负"实现行为说明"的责任。

## 1. plugin.json × strict mode 组合矩阵

依据：[Claude Code marketplace.json strict mode 官方说明](https://code.claude.com/docs/en/plugin-marketplaces#strict-mode) + `pluginLoader.ts` 行为观察。

| marketplace 条目 `strict` | `plugin.json` 状态 | 结果 |
| --- | --- | --- |
| `true`（默认）| 缺失 | ✅ 正常装载；用最小化 manifest 兜底（仅 `name` + 占位 `description`），其余字段来自 marketplace 条目 |
| `true` | 存在（合法 JSON）| ✅ 装载；`plugin.json` 是事实源，marketplace 条目字段（如 `description`）作为补充合并 |
| `true` | 存在但 JSON 解析失败 | ❌ 抛错，装载中止（Claude Code 原版）；⚠️ **双 SDK 实际选择容错**（见下方 note） |
| `false` | 缺失 | ✅ 装载；marketplace 条目即组件定义全集 |
| `false` | 存在且声明组件（commands/agents/hooks 等） | ❌ 冲突错误，装载失败 |
| `false` | 存在但不声明组件（仅元数据如 description）| ✅ 装载；marketplace 条目是组件权威，`plugin.json` 仅作元数据补充 |

!!! note "plugin.json 解析失败：Claude Code fatal vs 双 SDK 容错（Module D 决策点）"

    Claude Code 原版对「`plugin.json` 存在但解析失败」是 fatal。但两个 A2C 参考实现都选择了**容错降级**：

    * python-sdk `skills/manifest.py read_plugin_metadata`：best-effort，损坏 → `logger.warning("plugin manifest ignored")` + 返回 `{}`；注释明言「损坏不致命（SKILL/server 由路径推导）」
    * rust-sdk `manifest.rs` `read_plugin_metadata`：同样 WARN + 空 map

    Module D 若照本文档复刻 Claude Code 的 fatal 行为，会与 Computer 侧（同为参考实现的 SDK）**行为分叉**——一个损坏的 `plugin.json` 在 Robot 侧装载中止、在 Computer 侧却正常装载。建议 Module D 对齐双 SDK 的容错姿态；若坚持 fatal，需推动 SDK 同步修改（另行决策）。

## 2. plugin.json 缺失时的兜底逻辑

参考 Claude Code `pluginLoader.ts:1147-1160`：

```ts
if (!(await pathExists(manifestPath))) {
  return {
    name: pluginName,             // ← 来自 marketplace entry.name
    description: `Plugin from ${source}`,
  }
}
```

**关键观察**：

* **不读目录名做 fallback name** —— 用的是 marketplace entry 里的 `name`
* **不扫描 `commands/` / `agents/` / `skills/` 等目录来"猜"插件结构** —— 必须在 marketplace 条目或 `plugin.json` 显式声明
* **不存在"按约定自动发现"机制** —— Claude Code Plugin 不是 convention-over-configuration

> Module D 实施建议：复刻这套行为。SKILL 目录扫描可能是个例外（[SKILL 协议](../skill/protocol.md) §2 描述了 SKILL 目录约定），但 plugin 级别的组件发现应该按显式声明走，避免引入隐式约定。

## 3. plugin.json schema 字段必填性

| 字段 | 必填 | 说明 |
| --- | --- | --- |
| `name` | ✅ | kebab-case 标识符（即使 `plugin.json` 缺失，`name` 也可来自 marketplace 条目）|
| `version` | ❌ | semver；省略则用 git commit SHA。**优先级**（SDK `resolve_plugin_version` 实测）：marketplace 条目 `version` > `plugin.json` `version` > git commit SHA |
| `description` | ❌ | |
| `author` | ❌ | `{name: string, email?: string}` |
| `dependencies` | ❌ | 依赖其他 plugin |
| `commands` / `agents` / `skills` | ❌ | 组件路径或映射 |
| `hooks` | ❌ | hook 配置 |
| `mcpServers` / `lspServers` | ❌ | MCP/LSP 服务声明 |
| `userConfig` | ❌ | 启用时弹出的配置项 |
| `channels` | ❌ | MCP channel（助手模式） |

**只有 `name` 是必填** —— 但即使 `plugin.json` 缺失，`name` 也可由 marketplace 条目提供（见 §2 兜底逻辑）。

## 4. strict 模式真实语义

参考 Claude Code `pluginLoader.ts:2451-2706`：

| 模式 | 事实源 | 冲突处理 |
| --- | --- | --- |
| `strict: true`（默认）| `plugin.json` | 不冲突；marketplace 条目字段作为默认值合并 |
| `strict: false` | marketplace 条目 | 若 `plugin.json` 也声明 commands/agents/hooks，**触发冲突错误**（`pluginLoader.ts:2693-2706`）|

**适用场景**：

* `strict: false` 适合 "marketplace 运营方完全控制 plugin 元数据，作者只提供文件" 的 curator 场景
* 其他场景建议保持 `strict: true`（更宽松、更容错）

## 5. 实操建议（作者视角）

| 场景 | 建议 |
| --- | --- |
| 自己发 plugin 给用户直接 install | 写 `plugin.json`，至少含 `name` + `description` + `version` |
| 进官方 / 第三方 marketplace 让别人引用 | `plugin.json` 可省（marketplace 条目已经写了 `name`），**但强烈建议保留** —— 这样仓库本身被人 clone 当本地 plugin 时也能用 |
| 避免 `strict: false` | 除非是 marketplace 运营方需要锁死元数据，否则 `strict: true` 更宽松、容错性更好 |

## 6. 给 Module D 实施工程师

复刻这套行为时的关键决策点：

1. **plugin.json 缺失兜底**：用 marketplace 条目 `name` + 占位 description，**不**从目录名兜底
2. **冲突检测**：`strict: false` 时若发现 `plugin.json` 声明了任何组件字段（commands/agents/hooks/skills/mcpServers/lspServers），立即报错而非静默合并 —— 防止"以为生效但实际被覆盖"
3. **JSON 解析错误**：Claude Code 原版对「`plugin.json` 存在但解析失败」一律视为致命错误，**不**降级到缺失兜底（避免掩盖作者的语法 bug）。⚠️ **双 SDK 实际选择容错降级**（WARN + 空 manifest，SKILL/server 由路径推导）——见 §1 note，Module D 需择一并对齐 Computer 侧
4. **不引入按约定的目录自动发现**：组件目录必须显式声明，保持与 Claude Code 行为一致；这样作者跨阵地切换不会遇到"在 A 端能跑、在 B 端找不到组件"的怪现象

## 7. Marketplace source vs Plugin source —— schema 层 disjoint union

主规范 [protocol.md §5.3](protocol.md) 给出了两套 source 的对照表。本节补充源码层观察。

### 7.1 Claude Code 原生 schema（参考实现，全集）

参考 Claude Code `schemas.ts:906-1161`：

```ts
// Marketplace source —— 用户加 marketplace 时用的
MarketplaceSource:  url | github | git | npm | file | directory
                  | hostPattern | pathPattern | settings

// Plugin source —— marketplace.json 里 plugin 条目的 source 字段
PluginSource:       relative | npm | pip | url | github | git-subdir
```

### 7.2 TFRobotServer 实施子集（缩减后）

全部是 Git 引用，`github` / `cnb` 是简写糖到不同 host。理由见 [Marketplace 规范 §11](protocol.md)：

```
MarketplaceSource:  url | github | git | cnb              ← 4 类
PluginSource:       relative | url | github | git-subdir | cnb   ← 5 类
```

> **Computer 侧实施状态**：A2C SDK 的 Marketplace source 目前收敛为仅 `{type: "git", url}`（无 ref/sha、无简写糖，`marketplace_clone_url` 硬性校验）——上表 4 类是 Robot 侧（Module D）契约；作者配置若直接进 Computer 侧，`github` / `cnb` 对象形态会被拒绝（另见主规范 [protocol.md §5.3](protocol.md) 标注）。

简写糖归一化（D3 Loader 实施）：

| 输入 | 归一化为 |
| --- | --- |
| `{source: "github", repo: "owner/repo"}` | `https://github.com/owner/repo.git` |
| `{source: "cnb", repo: "group/project"}` | `https://cnb.cool/group/project.git` |

**关键事实**：

* 两类 source **没有交集**（disjoint union）
* `git-subdir` 仅 PluginSource 一侧；marketplace 不能用 git-subdir 添加
* `git` 仅 MarketplaceSource 一侧；plugin 同语义用 `url`（discriminator 不同强制区分）
* 相对路径（裸字符串）仅 PluginSource 一侧
* `cnb` 在**两侧都存在**，字段定义独立（MarketplaceSource `cnb` 比 PluginSource `cnb` 多 `path`/`sparsePaths`）
* 加载器靠 `source.source` 字段（union discriminator）判定，**不是递归嵌套** —— marketplace 不可能在 schema 层"嵌套另一个 marketplace"

## 8. git-subdir 加载链路

参考 Claude Code `pluginLoader.ts:712-800` / `:979`：

```
git clone --filter=tree:0  --sparse-checkout=<path>     ← 部分克隆，只拉目标子目录
    ↓
mv  <cloneDir>/<path>  →  targetPath                    ← 仅保留目标子目录
    ↓
丢弃整个临时 clone 目录                                  ← pluginLoader.ts:712
    ↓
读取 targetPath/.claude-plugin/plugin.json              ← pluginLoader.ts:979（注意是 plugin.json，不是 marketplace.json）
```

加载 `adobe-for-creativity`（举例）时：

* Claude Code 临时部分克隆 `adobe/skills`
* 只保留 `plugins/creative-cloud/adobe-for-creativity/` 一个目录
* **不读** Adobe repo 根目录的 `marketplace.json`
* 在子目录里找 `plugin.json` 当作 plugin 元数据（或在 §1 矩阵下走 strict mode 行为）

## 9. 双重身份 repo 的加载独立性

一个 repo 可同时承担两种身份：

* **身份 A — 自营 marketplace**：根目录 `.claude-plugin/marketplace.json` 让粉丝直接 `/plugin marketplace add` 订阅
* **身份 B — Plugin 提供方**：子目录被第三方 marketplace 用 `git-subdir` 引用

两条加载路径**不共享 clone**：

| 视角 | 装载器行为 |
| --- | --- |
| 用户加 adobe/skills 为 marketplace | 整个 repo clone 到 `~/.claude/plugins/marketplaces/skills/`，读根 marketplace.json，列出 Adobe 全套 plugin |
| 用户从官方市场装 adobe-for-creativity | 独立 sparse clone adobe/skills，只提子目录，作为单个 plugin 安装；不读 adobe repo 根的 marketplace.json |

**代价**：磁盘上同一 repo 可能被独立拉取两次。

**收益**：cache 模型简单 —— 避免单一 cache 既要服务 marketplace 视图又要服务 plugin 视图的两难。Module D 实施时建议复刻这个分层，不引入"共享 clone"优化以免引入耦合 bug。

## 10. 运行时状态层：install / enable 分离（v0.3.0 语义）

v0.2.x 是「**装即活跃**」：`install` 一个 plugin，其 SKILLs 立即进入投影。v0.3.0 把 **install（安装）** 与 **enable（启用）** 分离为两个可分别停留的状态——`install` 只安装、**不激活**，`enable` 才激活。这是一次**破坏性行为变更**（权威：[A2C-SMCP v0.3.0 迁移指南](https://doc.turingfocus.cn/a2c-smcp/latest/migrations/v0.3.0-plugin-install-enable-separation/) / runtime-contract §2.4）。

### 10.1 状态模型

两个正交、独立生命周期的维度：

- **是否安装**——全局一次，承载于 `installedPlugins`（已安装 `<plugin>@<marketplace>` 集合）。
- **本 scope 是否启用**——per-scope，承载于 `enabledPlugins`（`<plugin>@<marketplace>` → bool）。

组合出三态：

| 状态 | `installedPlugins` | `enabledPlugins`（合并后） | 投影 |
|---|---|---|---|
| `available` | 不含 | —— | 无 |
| `installed_disabled` | 含 | absent 或 `false` | **无**（惰性） |
| `installed_enabled` | 含 | `true` | skills + bundled server **一并** |

`enabledPlugins[id]` 三态（scope 合并 user < project < local）：absent = 无意见、继承上层（无任何 scope 置 `true` 即不激活）；`true` = 启用；`false` = 显式禁用（覆盖上层 `true`）。

### 10.2 行为对照

| 动作 | v0.2.x（装即活跃） | v0.3.0（分离） |
|---|---|---|
| `install` | 写账本 + skills 立即 active（server 需 hooks 才挂） | 写 `installedPlugins` + 物化；**不激活**、不写 `enabledPlugins` → `installed_disabled` |
| `enable` | 补挂 bundled server | 写 `enabledPlugins=true` + **原子**投影 skills 与 server；失败回滚 `installed_disabled` |
| `disable` | 反激活 | 写 `enabledPlugins=false` + 移除投影；保留 installation |
| `uninstall` | 删账本 + teardown | 删 `installedPlugins` + 清 `enabledPlugins` 条目 + teardown |
| boot | 账本存在即恢复 active | 活跃集 = 已安装 ∧ 本 scope 启用；`installed_disabled` 恢复为惰性 |
| 账本删除 | 可能丢安装集 | **无损**：从 `installedPlugins` 重物化 |

### 10.3 相关持久化与信任记录

- **物化记录**：`known_marketplaces.json`（已添加 marketplace 目录）/ `installed_plugins.json`（已安装 plugin 集）。旧「账本」降级为可从 `installedPlugins` 重建的**纯派生缓存**——MUST NOT 手编、MUST NOT 作为权威。
- **trustedMarketplaces**：`marketplace add` 有 trust 确认并持久化——marketplace 源被视作可执行信任边界（SKILL 脚本执行权限继承自安装来源）。
- **一次性迁移**（v0.2.x → v0.3.0）：既有用户的账本有记录、`enabledPlugins` 无条目——在 v0.2.x 下是 active，在 v0.3.0 下（absent = 未启用）会「熄灯」。SDK 升级 MUST 提供一次性迁移：把每个「账本已安装」的 plugin 写入 `installedPlugins`，并在**记录该安装的 scope** 写 `enabledPlugins=true`（迁为 `installed_enabled`）。注意 `enabledPlugins` 是 per-scope，project / local 的 `false` 会覆盖 user 的 `true`——迁移 MUST 写在能确保合并后为 `true` 的 scope。此迁移只跑一次。

> 对第三方作者的实际意义：`/plugin install <plugin-name>@<marketplace-name>`（§2.3）只安装**不激活**——「装了为什么没生效」的答案通常是**尚未 `enable`**，而不是配置错误。

## 11. 设计后果（作者视角）

理解上述两套 schema + 加载独立性后，作者可获得的能力：

1. **一个 repo 多用**：根目录是自家 marketplace（让订阅者直接装），子目录被第三方 curator 引用 —— 两种身份并行不悖
2. **Curator marketplace 不必托管代码**：curator 仅在自己的 `marketplace.json` 里写 plugin 指针（含 `sha` 锁版本），实际内容归属上游 plugin 作者
3. **粒度精确**：可以从一个 monorepo 挑某个子目录作为 plugin 引用，不强迫作者拆 repo
4. **sha 锁版本**：curator 可"快照" plugin 某个 commit，上游作者后续改动不会立即影响 curator 用户

→ "marketplace 嵌套 marketplace" 是**错觉** —— `marketplace.json` 里的 plugin 条目从来都是指向 plugin 源，从不指向另一个 marketplace；只是 plugin 源恰好可以是某个 repo 的子目录，而那个 repo 本身**碰巧**也对外提供 marketplace 接口而已。两个身份在 schema 层与加载链路层都完全隔离。

## 12. 参考

* [Claude Code Plugin Marketplaces 官方文档](https://code.claude.com/docs/en/plugin-marketplaces) —— §strict mode 章节
* [Claude Code Plugin Reference](https://code.claude.com/docs/en/plugins) —— plugin manifest schema
* Claude Code 源码（观察依据）：`src/utils/plugins/pluginLoader.ts`、`src/utils/plugins/schemas.ts`
