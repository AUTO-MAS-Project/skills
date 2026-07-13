# 插件声明式 UI 指南

## 1. 定义与边界

声明式页面扩展的 renderer、Manifest 与宿主规则在本文说明；Schema 表单支持的全部控件、属性、校验、动作和已知差异见 [声明式 Schema UI 元素全集](schema-ui-elements.md)。

AUTO-MAS 的插件声明式 UI 是一条“后端声明、前端宿主渲染”的扩展链：

1. 插件通过 `ctx.page.register()` 声明页面元数据；
2. 后端 `PageRegistry` 校验、排序并记录页面归属；
3. `/api/plugins/get` 与 `PluginSystem` WebSocket 快照返回页面声明；
4. 前端根据 `renderer` 动态生成菜单和 Vue Router 路由；
5. `iframe` 或 `custom-element` 宿主加载插件实际界面。

它解决的是“插件如何向主应用声明一个页面”，不是一套任意组件 JSON DSL，也不是让插件直接修改主程序 Vue Router。插件配置 Schema 和脚本 Schema 表单属于另一类声明式表单，不应与页面扩展混为一谈。

关键实现：

- 页面模型与注册：`app/core/page_registry.py`
- 前端 Manifest 与资源补全：`app/plugins/frontend_extensions.py`
- 页面快照 API：`app/api/plugins.py`
- 动态路由：`frontend/src/router/pageDeclarations.ts`
- iframe 宿主：`frontend/src/views/PluginPageHost.vue`
- Custom Element 宿主：`frontend/src/views/PluginElementHost.vue`
- 前端资源加载：`frontend/src/plugin/pluginFrontendLoader.ts`
- 插件页面 API：`frontend/src/plugin/pluginAPI.ts`

## 2. 选择渲染模式

| 模式 | 适用场景 | 隔离性 | 宿主能力 | 推荐度 |
|---|---|---:|---:|---:|
| `custom-element` | 与 AUTO-MAS 视觉和交互紧密集成的插件 UI | 低，共享 DOM/页面环境 | 可直接使用 `window.pluginAPI`，可共享宿主 Vue runtime | 默认推荐 |
| `iframe` | 已有独立 Web UI、需要样式/运行环境隔离 | 高 | 不自动继承宿主 JS 上下文 | 独立应用推荐 |
| `component` | 主程序内建页面 | 无隔离 | 完整主程序能力 | 插件不要使用 |

选择原则：

1. 新写、轻量、需要主应用上下文的插件页面，优先 `custom-element`；
2. 已有完整网页、依赖自己的路由/框架或必须隔离 CSS 时，使用 `iframe`；
3. 插件不能通过 `component` 指向自身 Vue 文件；该模式只解析 `PAGE_COMPONENTS` 中的主程序内建组件；
4. 只需要几个配置字段时，不要创建页面，直接使用插件配置 Schema；
5. 只需要脚本/用户配置时，使用 Script Adapter 的 Schema 编辑器，不创建重复菜单页。

## 3. 页面声明契约

### 3.1 最小 Custom Element 页面

```python
class Plugin:
    def __init__(self, ctx):
        self.ctx = ctx

    async def on_start(self) -> None:
        self.ctx.page.register(
            id="example-dashboard",
            path="/example-dashboard",
            title="示例面板",
            menu_label="示例面板",
            icon="app",
            renderer="custom-element",
            element_tag="auto-mas-example-dashboard",
            section="main",
            order=100,
        )
```

`frontend_plugin` 通常不必手写。`PageFacade` 会根据当前插件名补全，后端快照也会根据页面 `source=plugin:<instance_id>` 与运行实例再次推断。

### 3.2 最小 iframe 页面

```python
async def on_start(self) -> None:
    self.ctx.server.http("/example/ui", self.render_page, methods=("GET",))
    self.ctx.page.register(
        id="example-web",
        path="/example-web",
        title="示例网页",
        menu_label="示例网页",
        icon="app",
        renderer="iframe",
        url="/plugin/example/ui",
        section="main",
        order=110,
    )
```

`url` 的处理规则：

- `http://`、`https://` 或 `//`：按原地址加载；
- `/plugin/example/ui`：拼接 AUTO-MAS 后端基址；
- `plugin/example/ui`：自动补 `/` 后拼接后端基址；
- 空值：宿主显示“插件页面缺少入口”。

### 3.3 批量声明

```python
self.ctx.page.register_many(
    [
        {
            "id": "example-main",
            "path": "/example",
            "title": "示例",
            "menu_label": "示例",
            "renderer": "custom-element",
            "element_tag": "auto-mas-example",
            "section": "main",
            "order": 100,
        },
        {
            "id": "example-debug",
            "path": "/example-debug",
            "title": "示例调试",
            "menu_label": "示例调试",
            "renderer": "custom-element",
            "element_tag": "auto-mas-example-debug",
            "section": "dev",
            "order": 100,
            "dev_only": True,
        },
    ]
)
```

一个前端 Manifest 可以声明多个 Custom Element。多页面插件应在每个页面声明中显式设置不同的 `element_tag`。

## 4. `PageDeclaration` 字段说明

| 字段 | 必填 | 含义与约束 |
|---|---:|---|
| `id` | 是 | 页面稳定 ID；非空，全局唯一；动态路由名默认为 `page:<id>` |
| `path` | 是 | 前端路由；自动规范为单个前导 `/`，去除重复与尾部 `/` |
| `title` | 是 | 页面标题，也作为宿主元素的 `title` 属性 |
| `menu_label` | 是 | 侧边菜单文案 |
| `icon` | 否 | 主前端图标键；未知键回退为 `app` |
| `component` | 否 | 默认 `PluginPage`；仅 `component` renderer 使用 |
| `renderer` | 否 | `component`、`iframe`、`custom-element`，默认 `component` |
| `url` | iframe 是 | iframe 页面入口 |
| `frontend_plugin` | 通常否 | 前端资源所属插件 ID；通常由实例归属自动补全 |
| `element_tag` | 条件必填 | Custom Element 标签；Manifest 仅含一个元素时可自动推断 |
| `entry_asset_url` | 不应手填 | 后端根据 Manifest 生成的入口 URL |
| `style_asset_urls` | 不应手填 | 后端根据 Manifest 生成的样式 URL |
| `manifest_version` | 不应手填 | 用于生产资源缓存键和查询参数 |
| `section` | 否 | `main`、`bottom`、`dev`；默认 `main` |
| `order` | 否 | 同区域内升序排列，默认 `1000` |
| `visible` | 否 | 是否进入菜单，默认 `True`；不代表路由权限 |
| `dev_only` | 否 | 仅开发环境显示菜单，默认 `False`；不代表安全隔离 |
| `source` | 自动 | `PageFacade` 写入 `plugin:<instance_id>`，用于归属和清理 |

### ID 与路径命名

推荐给插件页面增加插件前缀：

```text
id:   example-dashboard
path: /example/dashboard
```

不要使用 `home`、`scripts`、`plans`、`plugins`、`plugins-market`、`queue`、`scheduler`、`history`、`tools`、`settings` 等内建 ID 或路径。

页面冲突由注册表按来源优先级处理，插件声明优先于主程序声明。这是平台覆盖机制，不应被普通插件用于替换核心页面。冲突只会记录警告，未必让插件启动失败，因此开发者必须主动检查 `page_errors`。

### 菜单可见性不是权限

`visible=False` 只是不生成菜单项，动态路由仍可能存在；`dev_only=True` 也主要控制开发菜单显示。敏感页面必须在后端接口实施鉴权、权限或环境校验，不能依赖隐藏菜单。

## 5. Custom Element 前端包

### 5.1 目录结构

```text
plugins/example/
├─ pyproject.toml
├─ frontend/
│  ├─ manifest.json
│  ├─ index.js
│  └─ index.css
├─ frontend-src/                    # 可选，开发源码
│  ├─ plugin.frontend.dev.json
│  └─ src/...
└─ src/example/
   └─ plugin.py
```

`frontend/` 必须是已安装 Python 包资源的一部分。在 `pyproject.toml` 中配置 package data，确保 wheel 安装后仍包含 Manifest、JS 和 CSS；仅本地目录存在但未打包不算完成。

### 5.2 生产 Manifest

`frontend/manifest.json`：

```json
{
  "version": 1,
  "renderer": "custom-element",
  "entry": "frontend/index.js",
  "style": ["frontend/index.css"],
  "elements": [
    { "tag": "auto-mas-example-dashboard" },
    { "tag": "auto-mas-example-debug" }
  ]
}
```

字段规则：

- `version`：大于等于 1；每次需要前端强制更新时递增；
- `renderer`：当前只能是 `custom-element`；
- `entry`：必填，入口 ES module；
- `style`：字符串或字符串数组均可，后端归一为数组；
- `elements`：页面可用标签列表；
- `entry` 与 `style` 必须以 `frontend/` 开头；
- 路径禁止绝对路径、空段、`.`、`..`；
- 模型 `extra="forbid"`，未知字段会使 Manifest 无效。

后端会把资源转换为：

```text
/api/plugins/assets/<plugin_id>/frontend/index.js?v=<version>
/api/plugins/assets/<plugin_id>/frontend/index.css?v=<version>
```

不要在页面声明中自行构造这些 URL。

### 5.3 Custom Element 入口

原生 JavaScript 最小示例：

```javascript
class ExampleDashboard extends HTMLElement {
  connectedCallback() {
    this.render()
  }

  disconnectedCallback() {
    this.cleanup?.()
  }

  async render() {
    const context = window.pluginAPI.getPageContext()
    const response = await window.pluginAPI.call('example/status')
    this.innerHTML = `
      <section class="example-dashboard">
        <h2>${context?.title ?? '示例面板'}</h2>
        <pre>${escapeHtml(JSON.stringify(response, null, 2))}</pre>
      </section>
    `
  }
}

if (!customElements.get('auto-mas-example-dashboard')) {
  customElements.define('auto-mas-example-dashboard', ExampleDashboard)
}
```

注意：

- 标签名必须包含连字符，建议 `auto-mas-<plugin>-<page>`；
- 入口脚本必须调用 `customElements.define()`；加载器最多等待 8 秒；
- HMR 或重复加载时先检查 `customElements.get()`，避免重复定义抛错；
- `connectedCallback` 可能多次执行，初始化必须幂等；
- `disconnectedCallback` 必须清理定时器、订阅、观察器和全局事件；
- 插件内容来自自身包也不要直接把未转义数据写入 `innerHTML`；优先 DOM API 或框架模板转义。

### 5.4 使用 Vue

宿主加载入口前会设置：

```javascript
window.__AUTO_MAS_PLUGIN_VUE__
```

该值是主前端当前使用的 Vue runtime。插件可在构建时将 `vue` external 化，并从该全局获取 runtime，避免把第二份 Vue 打进插件包。具体 bundler 配置应与插件模板保持一致。

约束：

- 不假定可导入主前端内部模块别名，如 `@/stores/...`；
- 不依赖未声明的 Ant Design Vue 私有上下文；
- 不修改宿主 Vue app、router、Pinia 或全局 prototype；
- 卸载 Custom Element 时调用 Vue app 的 `unmount()`；
- 插件 CSS 必须带插件前缀或使用 Shadow DOM，防止污染主应用。

Vue Custom Element 的概念示例：

```javascript
const { createApp, h } = window.__AUTO_MAS_PLUGIN_VUE__

class ExampleElement extends HTMLElement {
  connectedCallback() {
    if (this.app) return
    this.app = createApp({ render: () => h('div', { class: 'example-root' }, '示例页面') })
    this.app.mount(this)
  }

  disconnectedCallback() {
    this.app?.unmount()
    this.app = null
  }
}

if (!customElements.get('auto-mas-example')) {
  customElements.define('auto-mas-example', ExampleElement)
}
```

## 6. Custom Element 宿主能力

### 6.1 元素属性

`PluginElementHost.vue` 创建元素时提供：

```html
<auto-mas-example
  title="页面标题"
  data-page-id="example-dashboard"
  data-plugin-id="example"
></auto-mas-example>
```

页面切换或声明变化可能销毁并重建元素。不要把关键状态只保存在元素实例中；需要持久化时调用后端或插件缓存。

### 6.2 `window.pluginAPI`

类型契约：

```typescript
interface PluginAPI {
  call(path: string, payload?: unknown): Promise<unknown>
  subscribe(topic: string, handler: (message: WebSocketMessage) => void): () => void
  getPageContext(): PluginPageContext | null
}
```

#### 调用后端

```javascript
const result = await window.pluginAPI.call('example/items')
const saved = await window.pluginAPI.call('example/items/save', { name: 'demo' })
```

路径与方法规则：

- 相对路径 `example/items` → `/plugin/example/items`；
- 绝对路径 `/plugin/example/items` → 后端基址 + 原路径；
- 完整 `http(s)` URL 原样调用；
- `payload === undefined` 使用 GET；传入任意 payload 使用 POST + JSON；
- 当前 API 不支持任意 HTTP method、自定义 headers、取消请求或自动 unwrap `data`；需要这些能力时应先扩展公共契约，而不是插件各自读取宿主私有对象。

`pluginAPI.call()` 只按 HTTP 状态判断成功。若插件 endpoint 使用 HTTP 200 返回 `{code: 400}`，调用不会自动抛错；建议插件让业务错误与 HTTP 状态一致，或前端显式检查 `code/status`。

#### 订阅主 WebSocket 主题

```javascript
const unsubscribe = window.pluginAPI.subscribe('ExampleTopic', message => {
  console.log(message.type, message.data)
})

this.cleanup = () => unsubscribe()
```

该能力订阅的是 AUTO-MAS 主 WebSocket topic，不是 `ctx.server.websocket()` 注册的任意原生 WS 路径。插件自定义 WS 路由需要自行基于后端地址建立 WebSocket，或扩展统一插件 API。

#### 获取页面上下文

```javascript
const context = window.pluginAPI.getPageContext()
```

返回：

```typescript
{
  pageId: string
  path: string
  title: string
  renderer: string
  source: string
  pluginId: string | null
  elementTag: string | null
}
```

上下文是宿主当前页面的模块级快照，不是响应式对象。不要缓存后永久假设不变；需要时重新读取。多个 Custom Element 页面并行常驻并非当前模型的目标，不要把它当跨页面全局状态。

## 7. iframe 页面

### 7.1 特性

iframe 宿主设置：

```html
sandbox="allow-scripts allow-forms allow-popups allow-modals allow-downloads allow-same-origin"
```

未授予顶层导航、父页面 DOM 控制等额外权限。iframe 有独立 `window`，不会自动获得父窗口的 `window.pluginAPI` 或 `__AUTO_MAS_PLUGIN_VUE__`。

### 7.2 与后端通信

同源 iframe 可直接调用后端相对路径：

```javascript
const response = await fetch('/plugin/example/status')
```

但在桌面前端与后端端口不同的环境中，iframe 页面自身的 origin 取决于 `url`。稳妥做法是让 iframe HTML 由 `/plugin/<name>/...` 提供，并从同一后端 origin 调用插件 API。

外部 URL iframe：

- 不能假定可访问本机后端；
- 必须处理 CORS、认证和网络不可用；
- 不应把本地敏感数据放入查询参数；
- 不应依赖 `allow-same-origin` 作为信任证明；外部内容仍是第三方内容。

### 7.3 与宿主通信

当前宿主没有定义 iframe `postMessage` 协议。如确需双向通信，应先在平台层设计：

- 明确消息 envelope、允许 origin、请求 ID 与超时；
- 父窗口严格校验 `event.origin` 和消息结构；
- 不让插件直接访问 `window.parent` 内部状态；
- 在公共类型和文档中稳定该契约。

不要在单个插件里私自约定并修改主宿主。

## 8. 开发模式

### 8.1 开发 Manifest

当 `AUTO_MAS_DEV=1` 时，后端优先读取：

```text
frontend-src/plugin.frontend.dev.json
```

使用本地 Vite 服务：

```json
{
  "version": 0,
  "renderer": "custom-element",
  "entry_url": "http://localhost:5174/src/main.ts",
  "style_urls": ["http://localhost:5174/src/style.css"],
  "elements": [{ "tag": "auto-mas-example" }],
  "command": "npm run dev"
}
```

或直接加载源码文件：

```json
{
  "version": 0,
  "renderer": "custom-element",
  "entry": "src/main.ts",
  "elements": [{ "tag": "auto-mas-example" }]
}
```

规则：

- `entry` 与 `entry_url` 至少提供一个；
- `entry_url`、`style_urls` 只允许 `http(s)` 的 `localhost`、`127.0.0.1`、`::1`；
- `entry` 禁止空段、`.`、`..`，后端转换为 Vite `/@fs/` URL；
- `command` 目前作为页面快照元数据提供，不应假设前端一定自动执行；
- 主前端 Vite 必须允许访问插件源码目录，否则 `/@fs/` 会被拒绝。

### 8.2 HMR 行为

插件文件变化会通过 `PluginSystem` 通知前端。前端对 `frontend_refresh` 的处理是刷新插件前端乃至整页，不是组件级状态保持 HMR。

加载器缓存：

- 生产入口键：`plugin + manifest_version + entry_url`；
- 开发入口键：`plugin + dev + entry_url`；
- CSS URL 加载后保留在 `<head>`；
- 已加载脚本不会因路由离开自动移除；
- `customElements` 注册在浏览器全局，不能注销。

因此：

- 生产前端变更应递增 Manifest `version`；
- 开发时不要依赖重新执行同一入口来替换已注册标签；
- 代码更新后通常接受页面刷新；
- CSS 文件名或 URL 不变时，开发服务必须正确处理缓存；
- 不要动态生成无限多标签名、入口 URL 或样式 URL，否则会累积全局资源。

## 9. 生命周期与清理

页面声明归属于 `plugin:<instance_id>`。插件加载失败、停止、卸载或重载时，Loader 会按 owner 清理页面注册、声明式服务、监听器和脚本类型；插件通常无需手动调用 `ctx.page.unregister_all()`。

仍应保证：

- `on_start()` 可重复执行，不留下重复的插件内部资源；
- 页面 ID 和 path 在重载前后稳定；
- 插件前端元素在 `disconnectedCallback()` 中清理自身资源；
- `pluginAPI.subscribe()` 返回的退订函数一定被调用；
- iframe 自己负责关闭其定时器、连接和页面级资源；
- 停止插件后，前端下一次页面快照会移除动态路由；若用户正停留在该页，应允许主布局回退或显示不可用态；
- 不依赖删除 `<script>`、`<link>` 或 Custom Element 定义来完成热卸载，因为当前加载器不会这样做。

## 10. 安全与样式注意事项

### Custom Element

- 与主应用同一 JavaScript realm，视为受信任代码；插件可访问页面全局对象；
- 不加载未经审查的远程入口脚本；生产 Manifest 只允许插件包内资源；
- CSS 默认是全局的，使用插件前缀、CSS Modules 产物或 Shadow DOM；
- 不覆盖 `html`、`body`、`.ant-*` 等全局选择器；
- 不读取或修改主应用未公开 DOM、store 和内部模块；
- 渲染外部数据时防止 XSS；
- 不在前端保存 token、密码或解密后的长期敏感信息。

### iframe

- sandbox 是能力限制，不是内容可信证明；
- 外部页面可能追踪用户或泄露上下文，默认优先本地插件页面；
- 任何 `postMessage` 都必须校验 origin；
- 后端业务接口仍需校验输入与权限，不能以“只能从 iframe 调用”为安全边界。

### 路由和菜单

- `visible`、`dev_only` 不是访问控制；
- `path` 不携带秘密或敏感配置；
- 不覆盖核心页面；
- 菜单标签应简短稳定，页面标题可更完整；
- 未知图标会回退，不要依赖未注册图标名展示关键语义。

## 11. 常见失败与定位

### 页面未出现在菜单

检查：

1. 插件实例是否启用并成功进入 `on_start()`；
2. `/api/plugins/get` 的 `pages` 是否包含声明；
3. `page_errors` 是否有 ID/path 冲突或 Manifest 错误；
4. `visible` 是否为 `True`；
5. `dev_only` 页面是否运行在开发环境；
6. `section` 是否为受支持值。

### 路由存在但页面空白

iframe：检查 `url`、响应状态、CSP/CORS、浏览器控制台和 iframe Network。

Custom Element：检查：

1. `frontend_plugin` 是否正确；
2. `frontend/manifest.json` 是否被打入 Python 包；
3. `entry_asset_url` 请求是否 200 且 MIME 可作为模块加载；
4. 入口脚本是否抛异常；
5. `customElements.define()` 标签是否与 `element_tag` 完全一致；
6. 标签是否在 8 秒内注册；
7. Manifest 多元素时是否显式声明 `element_tag`。

### “入口已加载，但 custom element 未注册”

通常是：

- 构建产物只导出了组件但未执行 `customElements.define()`；
- 标签拼写或大小写不一致；
- 入口执行前异常；
- 把 Vue 当成已安装依赖，但实际 external/global 配置不匹配；
- 开发入口指向了普通应用 main，而非 Custom Element 注册入口。

### 更新后仍显示旧页面

检查：

- 生产 Manifest `version` 是否递增；
- 浏览器是否仍缓存同一 URL；
- Custom Element 是否因全局已注册而无法覆盖；
- 开发态是否触发 `frontend_refresh` 与页面刷新；
- CSS URL 是否变化或开发服务器是否发出正确缓存头。

## 12. 完整实现步骤

1. 选择 `custom-element` 或 `iframe`，说明选择理由；
2. 在插件 `on_start()` 中通过 `ctx.page.register()` 声明稳定 ID/path；
3. Custom Element：创建生产 Manifest、入口、样式并配置 package data；
4. iframe：通过 `ctx.server.http()` 或稳定外部 URL 提供页面；
5. 将数据接口注册在 `/plugin/<plugin>/...`；
6. 前端只使用公开的 `window.pluginAPI` 或标准 Web API；
7. 实现元素/页面卸载清理；
8. 启动本地后端与主前端，验证菜单、路由、加载、接口、异常态；
9. 禁用、重载、卸载插件，验证声明与动态路由被移除；
10. 构建 wheel 后再次安装验证，确保前端资源确实被打包。

## 13. 提交前检查清单

### 声明

- [ ] 页面 `id`、`path` 带插件命名空间且不覆盖核心页面；
- [ ] `title`、`menu_label`、`icon`、`section`、`order` 符合产品展示；
- [ ] renderer 选择有依据，未用 `component` 加载插件代码；
- [ ] `visible`、`dev_only` 未被误作权限控制；
- [ ] 插件重载后声明稳定且不重复。

### Custom Element

- [ ] Manifest 仅含支持字段，资源路径均位于 `frontend/`；
- [ ] Manifest `version` 与发布缓存策略一致；
- [ ] `elements` 与 `customElements.define()`、页面 `element_tag` 一致；
- [ ] 多元素 Manifest 的每个页面显式选择标签；
- [ ] JS/CSS/Manifest 已进入 wheel package data；
- [ ] 入口幂等，元素卸载会取消订阅并释放资源；
- [ ] CSS 不污染主应用；外部数据经过安全渲染；
- [ ] 只依赖 `pluginAPI` 与公开宿主契约。

### iframe

- [ ] URL 在桌面环境中可达；
- [ ] sandbox 能力足够但未被不必要放宽；
- [ ] CORS、CSP、认证和离线场景已验证；
- [ ] 未假定 iframe 自动拥有 `window.pluginAPI`；
- [ ] 若使用 `postMessage`，平台契约与 origin 校验完整。

### 运行验证

- [ ] `/api/plugins/get` 返回页面且 `page_errors` 为空；
- [ ] 三个菜单区域及排序正确；
- [ ] 直接访问路由与菜单跳转都正常；
- [ ] 加载中、缺入口、网络失败、入口异常均有可理解提示；
- [ ] 后端接口成功与失败状态均正确呈现；
- [ ] 开发刷新、生产缓存、禁用、重载和卸载行为已实际验证。
