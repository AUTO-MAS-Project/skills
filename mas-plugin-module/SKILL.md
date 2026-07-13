---
name: mas-plugin-module
description: 用于新增、修改或审查 AUTO-MAS 插件模块，包括插件管理/市场 UI、插件页面扩展、插件声明与实例、声明式 HTTP/WS、Script Adapter、动态 Schema 表单及插件运行链。
---

# AUTO-MAS 插件模块

## 目标

统一插件从包声明、发现与实例化，到前端展示、服务暴露、脚本类型注册和生命周期执行的实现方式。优先复用现有注册表、宿主页面和 Schema 表单，不为单个插件复制平台能力。

## 适用范围

遇到以下任务时使用本 Skill：

- 修改插件管理、插件市场或插件实例操作 UI；
- 为插件声明菜单页面、iframe 或 Custom Element；
- 新增插件配置 Schema、HTTP/WS 服务、事件、服务依赖或生命周期；
- 使用 `ScriptAdapterPlugin` 注册动态脚本类型；
- 审查插件前后端声明是否完整、路由与数据是否一致。

若任务是适配一个外部自动化脚本项目，先用 `mas-script-specialized-adapter` 完成脚本架构问诊；本 Skill 只约束其插件平台接入部分。

涉及插件页面声明、iframe、Custom Element、前端 Manifest、宿主 API、开发模式或 UI 生命周期时，必须先阅读 [插件声明式 UI 指南](references/declarative-ui.md)。涉及 Schema 表单控件、字段属性、布局、校验、动态选项或动作时，必须阅读 [声明式 Schema UI 元素全集](references/schema-ui-elements.md)。

## 架构总览

### 后端职责

| 层 | 主要位置 | 职责 |
|---|---|---|
| 插件包 | `plugins/<plugin>/` | `pyproject.toml` Entry Point、插件类、默认实例、插件专属 Schema/运行逻辑 |
| 插件平台 | `app/plugins/` | 发现加载、实例生命周期、Context 门面、服务/事件/缓存、前端资源、声明式服务 |
| 管理 API | `app/api/plugins.py` | 插件发现、安装、实例增删改、启停、前端资源 |
| 服务网关 | `app/api/plugin_gateway.py` | 将插件声明的 HTTP/WS 统一分发到 `/plugin/**` |
| 页面注册 | `app/core/page_registry.py` | 校验、排序和注销主程序/插件页面声明 |
| 脚本注册 | `app/core/script_types.py` | 动态 `ScriptTypeProvider` 注册表与 owner 归属 |
| 宿主存储 | `app/models/plugin_script_config.py` | 用稳定容器保存动态脚本和用户配置 |

### 前端职责

| 表面 | 主要位置 | 模式 |
|---|---|---|
| 插件管理 | `frontend/src/views/Plugin.vue` | 固定宿主页面；管理实例、状态、配置与布局 |
| 插件市场 | `frontend/src/views/PluginMarket.vue` | 固定宿主页面；展示、安装和更新插件 |
| 插件菜单页 | `PluginPageHost.vue` / `PluginElementHost.vue` | 由页面声明选择 iframe 或 Custom Element |
| 插件脚本编辑 | `EditView/Script/PluginScriptEdit.vue` | 注册表 + Schema 驱动通用表单 |
| 插件用户编辑 | `EditView/User/PluginUserEdit.vue` | 注册表 + Schema 驱动通用表单 |
| 动态路由 | `router/pageDeclarations.ts` | 拉取声明、排序、按 renderer 生成路由 |

## 前端显示模式

### 1. 固定宿主页面

插件管理和市场是平台能力，保留在主前端中。插件只能提供数据和动作，不复制管理页面。

### 2. 插件页面声明

完整字段契约、renderer 选型、Manifest、宿主 API、开发模式和排错方法见 [插件声明式 UI 指南](references/declarative-ui.md)。

通过 `ctx.page.register(...)` 声明页面。核心字段如下：

```python
ctx.page.register(
    id="example-page",
    path="/example",
    title="示例页面",
    menu_label="示例",
    icon="app",
    renderer="iframe",  # component | iframe | custom-element
    url="/plugin/example/page",
    section="main",     # main | bottom | dev
    order=100,
)
```

约束：

- `id` 与 `path` 必须稳定且唯一，`path` 使用绝对路径；
- 插件通常选择 `iframe` 或 `custom-element`；`component` 仅能引用主程序已注册组件；
- `iframe` 必须提供 `url`；相对 URL 会指向后端地址；
- `custom-element` 必须能解析 `frontend_plugin`、`element_tag` 和前端资源；
- `section` 只使用 `main`、`bottom`、`dev`，顺序由 `order` 决定；
- 页面由实例 owner 注册，并在实例停止/卸载时注销；不要手写静态 Vue 路由。

### 3. Custom Element 前端包

生产资源放在插件 Python 包的 `frontend/`，声明 `frontend/manifest.json`：

```json
{
  "version": 1,
  "renderer": "custom-element",
  "entry": "frontend/index.js",
  "style": ["frontend/index.css"],
  "elements": [{ "tag": "auto-mas-example" }]
}
```

- `renderer` 当前仅支持 `custom-element`；
- `entry`、`style` 必须位于 `frontend/`，禁止绝对路径、空段、`.` 与 `..`；
- 开发清单为 `frontend-src/plugin.frontend.dev.json`，仅在 `AUTO_MAS_DEV=1` 生效；
- 开发 URL 仅允许 `localhost`、`127.0.0.1`、`::1`；
- 资源统一经 `/api/plugins/assets/...` 提供，不自行拼接本地文件 URL。

### 4. Schema 驱动脚本编辑

动态脚本类型默认使用 `PluginScriptEdit.vue` 与 `PluginUserEdit.vue`：

- `editor_kind="schema"` 使用通用 Schema 编辑器；
- `editor_kind="plugin:<key>"` 表示归属插件，可响应插件 HMR；
- `builtin:<kind>` 仅用于确有复杂交互、通用 Schema 无法表达的主程序专页；
- 图标、主题色、支持模式、文档 URL 和 Schema 均来自脚本类型 descriptor；
- Hub 的新建脚本、编辑脚本、新建用户、编辑用户四条路径都必须按同一 descriptor 路由；
- 不在前端硬编码新插件类型枚举，不手改 `frontend/src/api/` 生成文件。

## 插件包声明格式

### 包入口

插件使用 Python Entry Point：

```toml
[project.entry-points."auto_mas.plugins"]
example = "example.plugin:Plugin"
```

插件模块可声明默认实例：

```python
DEFAULT_INSTANCE = {
    "name": "示例插件",
    "enabled": True,
    "config": {},
}
```

系统插件可额外使用稳定 `id`、`system=True`、`locked=True`。`enabled` 必须是布尔值，`config` 必须是对象。

### 插件类与 Context

```python
class Plugin:
    def __init__(self, ctx):
        self.ctx = ctx

    async def on_start(self) -> None: ...
    async def on_stop(self, reason: str) -> None: ...
    async def on_unload(self) -> None: ...
```

插件只通过 `PluginContext` 使用受控能力：

- `ctx.config`：实例配置；
- `ctx.logger` / `ctx.log`：日志；
- `ctx.event`：事件；
- `ctx.service`：`provides`、`needs`、`wants` 服务依赖；
- `ctx.server`：HTTP、WS 与前端动作；
- `ctx.page`：页面声明；
- `ctx.runtime` / `ctx.runtime_api`：主程序授权能力；
- `ctx.cache`：插件缓存。

不要从插件直接修改主 FastAPI router、Vue router 或全局注册表私有状态。

## 配置与 Schema

普通插件配置由 `schema.py`、`schema.json` 或插件模块内 `Config` 提供，平台按 Pydantic 规则加载和校验。字段至少声明 `type`，按需使用：

- `required`、`nullable`、`description`；
- `constraints`；
- `size`：`1/1`、`1/2`、`1/3`、`2/3`、`1/4`、`3/4` 或语义尺寸；
- 选项统一归一为 `{ "label": ..., "value": ... }`。

已知结构用明确类型和分组；只在未知外部配置的兼容边界使用 `dict[str, Any]`。Schema 不执行文件、网络、进程或调度逻辑。

## 声明式 HTTP 与 WebSocket

插件通过 Context 注册：

```python
self.ctx.server.http("/example/items", self.list_items, methods=("GET", "POST"))
self.ctx.server.websocket("/example/events", self.on_message)
```

外部路径统一为 `/plugin/example/...`。

- `/api/plugins/**` 只负责平台级插件与实例管理；
- `/plugin/**` 只负责插件自行声明的业务服务；
- handler 接收 `PluginHttpRequest`，明确读取 `json`、`query`、`headers`；
- 返回值可为对象、列表、文本或 `PluginHttpResponse`；
- 业务 JSON 统一返回稳定的 `code`、`status`、`message`，数据放 `data`；
- 预期输入错误返回 4xx，内部异常记录完整日志，对外不泄漏堆栈；
- WS handler 可接收消息与 session；连接、消息、断开逻辑各自单一职责；
- 新接口不要复用 `/api/scripts/**` 的旧兼容端点。

## Script Adapter 声明

脚本型插件继承 `ScriptAdapterPlugin`，实现 `build_script_adapters()`：

```python
ScriptAdapterDefinition(
    type_key="Example",
    display_name="示例脚本",
    script_model=ExampleConfig,
    user_model=ExampleUserConfig,
    hooks_factory=ExampleHooks,
    supported_modes=("AutoProxy",),
    icon="General",
    editor_kind="schema",
    metadata={"source": "example"},
)
```

声明规则：

- `type_key` 全局唯一且稳定；`display_name` 面向用户；
- `script_model` 与 `user_model` 必须同时提供；或同时提供 `script_groups` 与 `user_groups`，两套方式不可混用；
- `supported_modes` 必须有对应 hooks 实现，不声明空能力；
- 兼容旧配置时显式声明 legacy class name 与迁移器；
- `on_start()` 以 `instance_id` 为 owner 注册，`on_stop()`、`on_unload()` 按 owner 注销；
- hooks 负责 `check`、`prepare`、`finalize`、`on_crash` 与各运行模式，不把执行循环放进 API。

## 动态脚本数据契约

动态脚本持久化统一使用：

- `PluginScriptConfig`：`Meta.PluginTypeKey`、`Info.Name`、`PluginData.Config`、`UserData`；
- `PluginUserConfig`：`Meta.PluginTypeKey`、`Info.Name`、`PluginData.Config`。

`PluginScriptConfig` 是稳定的存储容器，`Meta.PluginTypeKey` 才是真实业务类型。前端与后端都必须据此恢复 provider；禁止用容器类名冒充业务类型。表单态与 JSON 存储态通过 codec 转换，不让 UI 直接操作宿主 JSON 字符串。

## 端到端数据流

1. Loader 从 `auto_mas.plugins` Entry Point 发现插件包；
2. Manager 按实例配置创建 `PluginContext` 并启动插件；
3. 插件在 `on_start()` 注册页面、服务、事件或 Script Adapter；
4. 页面注册表生成前端菜单和动态路由；前端资源由插件资源 API 加载；
5. Script Adapter 构建 Schema artifacts 与 `ScriptTypeProvider`；
6. Scripts Hub 获取 descriptor，按 `editor_kind` 打开通用或专用编辑器；
7. 表单数据经脚本 API 与 codec 写入 `PluginScriptConfig` / `PluginUserConfig`；
8. 调度器从 `Meta.PluginTypeKey` 找到 provider，创建 manager 与 hooks 执行；
9. 停止或卸载插件时按实例 owner 注销脚本类型、页面、服务和路由。

## 修改顺序

1. 先判定功能属于普通插件、前端扩展还是 Script Adapter；
2. 定义插件包 Entry Point、默认实例及配置 Schema；
3. 实现插件类和最小生命周期；
4. 通过 Context 声明服务、页面和依赖；
5. 需要脚本类型时再声明 models、definition 与 hooks；
6. 默认复用现有 Host/SchemaForm，仅在交互确实无法表达时增加专页；
7. 检查启停/卸载清理、错误态、不可用态与兼容迁移；
8. 运行后端测试，并实际启动前端验证菜单、页面、四条脚本编辑路径和 HMR/重载行为。

## 审查清单

- [ ] Entry Point 能发现插件，`DEFAULT_INSTANCE` 字段合法且默认行为安全；
- [ ] 插件只经 `PluginContext` 使用平台能力，模块职责没有越界；
- [ ] 配置 Schema 类型明确、默认值稳定、无执行副作用；
- [ ] 固定管理 UI、iframe、Custom Element、Schema 表单的选型合理；
- [ ] 页面 `id/path` 唯一，renderer 所需字段齐全，资源路径安全；
- [ ] HTTP/WS 经 `ctx.server` 注册并使用 `/plugin/**`，管理 API 未混入插件业务；
- [ ] 响应与错误结构稳定，WS 生命周期完整，外部输入在边界校验；
- [ ] Script Adapter 模型成对、模式真实、owner 注册与注销对称；
- [ ] `PluginTypeKey` 在 API、前端、持久化与调度链中一致；
- [ ] 新建/编辑脚本与新建/编辑用户四条路径均可达；
- [ ] 未手改 OpenAPI 生成代码，未硬编码可由注册表提供的信息；
- [ ] 插件禁用、停止、卸载、HMR 和异常退出后无残留页面、路由、服务或类型；
- [ ] UI 已在真实运行环境验证，不只通过类型检查。

## 避免

- 不为单个插件复制插件管理、路由同步或 Schema 表单；
- 不让插件直接导入并修改主程序 router/registry 私有状态；
- 不把插件业务端点放进 `app/api/plugins.py`；
- 不用新的静态前端枚举限制动态 `type_key`；
- 不同时维护等价的旧 `/api/scripts/**` 与新 `/plugin/**` 接口；
- 不在只有一个调用点时创建无边界价值的 builder、manager 或 helper；
- 不在缺少 UI、持久化或运行消费者时提前加入占位字段。
