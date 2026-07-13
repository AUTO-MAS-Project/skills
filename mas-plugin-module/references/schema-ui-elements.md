# 声明式 Schema UI 元素全集

## 1. 范围

本文以 `frontend/src/components/SchemaForm.vue` 的实际渲染分支为准，列出 AUTO-MAS 声明式表单当前支持的全部元素、别名、属性和限制。

相关实现：

- 字段类型：`frontend/src/types/schemaForm.ts`
- 元素渲染：`frontend/src/components/SchemaForm.vue`
- 字段 DSL：`app/plugins/fields.py`
- 普通插件 Schema：`app/plugins/schema.py`
- Script Adapter 编译：`app/plugins/script_adapter_schema.py`
- 动作执行：`frontend/src/composables/useSchemaActionRunner.ts`

注意：最终下发给前端的 `type` 决定控件。`ui_type` 本身不参与 `SchemaForm` 分支判断；在 Pydantic `PluginField(..., ui_type="slider")` 中，它会被转换成最终 Schema 的 `type: "slider"`。

## 2. Schema 外层结构

### 分组结构

```json
{
  "groups": [
    {
      "key": "Info",
      "label": "基本信息",
      "fields": [
        { "key": "Info.Name", "type": "string", "label": "名称" }
      ]
    }
  ]
}
```

- 字段路径取 `field.key || field.name`；
- `Info.Name` 会读写嵌套对象；
- 空分组被过滤；
- 仅有一个分组时不显示分组标题，多个分组时显示。

### 扁平结构

```json
{
  "Name": { "type": "string", "label": "名称" },
  "Enabled": { "type": "boolean", "label": "启用" }
}
```

对象键自动成为字段 `key`，所有字段进入默认分组。

## 3. 元素总览

| 元素 | `type` 或触发条件 | 值类型 | 实际控件 |
|---|---|---|---|
| 动作按钮 | `button`、`action`，或存在 `action`/`button` | 不写配置 | Button |
| 自动完成文本 | 字符串类型 + 选项 + `allow_custom=true` | string | AutoComplete |
| 有序多选 | `multiselect` + `selection_mode=ordered` | array | 编号按钮组 |
| 多选 | `multiselect`，或 list + 选项 | array | Select multiple |
| 单选 | `select`，或非 list 字段带选项 | 任意选项值 | Select |
| 布尔 | `boolean`、`bool` | boolean | Switch |
| 路径 | `folder`、`file`、`path` | string | Input + 系统选择器 |
| 密码 | 字符串类型 + `format=password` | string | InputPassword |
| 多行文本 | 字符串类型 + `format=textarea` | string | Textarea |
| 普通文本 | `string`、`str`、`uuid`、`datetime`、`related-id`、`readonly` | string | Input |
| 滑动条 | `slider` | number | Slider + InputNumber |
| 数字 | `number`、`integer`、`int`、`float` | number | InputNumber |
| 列表编辑器 | `list`、`list[...]` 且无选项 | array | 可增删单列表格 |
| 键值表 | `key_value` | object | 可编辑键值表 |
| 动态表格 | `table` | array<object> | 可增行列的表格 |
| JSON | `json` | object | JSON Textarea |
| 回退文本 | 其他未知 `type` | string | Input |

渲染优先级很重要：动作、自动完成、有序多选、多选、单选、布尔、路径、字符串、滑动条、数字、列表、键值表、表格、JSON、回退文本。字段带选项时可能覆盖其原始类型的控件。

## 4. 通用字段属性

| 属性 | 类型 | 特性 |
|---|---|---|
| `key` | string | 字段路径，优先于 `name` |
| `name` | string | `key` 缺失时作为路径 |
| `group` | string | 字段归属；`Action` 有特殊页头行为 |
| `type` | string | 决定控件 |
| `label` | string | 标签首选值 |
| `title` | string | `label` 缺失时作为标签 |
| `description` | string | 标签仍缺失时作为标签，否则作为帮助文本 |
| `default` | unknown | 默认值，由生成/加载层使用 |
| `required` | boolean | 显示必填标记；空值阻止保存 |
| `readonly` | boolean | 禁用控件，不代表隐藏 |
| `hidden` | boolean | 不渲染字段 |
| `sensitive` | boolean | 元数据；当前不自动改成密码框 |
| `placeholder` | string | 文本和路径输入提示 |
| `help` | string | 帮助文案，优先于 description/examples |
| `examples` | array | 无 help/独立 description 时显示示例 |
| `size` | string | Grid 宽度 |
| `options` | array | 选择项，优先于 enum |
| `enum` | array | options 缺失时作为选择项 |
| `constraints` | object | 字符串、数字等校验约束 |
| `configurable` | boolean | 后端是否持久化；前端不会据此隐藏 |
| `icon` | string | 普通字段不显示；页头动作可使用 |
| `ui_type` | string | 前端接口保留字段；不直接决定渲染 |

标签回退顺序：`label -> title -> description -> key/name`。

帮助文案顺序：校验错误 → `help` → 与标签不同的 `description` → `examples`。

## 5. 全部元素及特性

### 5.1 动作按钮

触发条件：

- `type: "button"`；
- `type: "action"`；
- 任意 type 只要含有效 `action` 或 `button` 对象，也会被优先渲染为按钮。

```json
{
  "key": "Action.Check",
  "type": "button",
  "label": "检查环境",
  "configurable": false,
  "action": {
    "label": "立即检查",
    "path": "/plugin/example/check",
    "method": "POST",
    "payload": { "root": "{{formModel.Info.RootPath}}" },
    "refresh": true
  }
}
```

特性：

- 不参与字段值校验；
- `action` 优先于兼容字段 `button`；
- 按钮文案：`action.label -> 字段标签`；
- `path` 必填，缺失时显示警告；
- `method` 默认 POST；GET 把 payload 放 query，其余方法放 body；
- payload 支持递归 `{{context.path}}` 模板；
- `refresh=true` 在成功后重新加载页面数据；
- 返回对象中 `code != 200` 会抛出业务错误；
- 字段位于 `Action` 组，或路径以 `Action.` 开头时，会被提升到编辑页页头并从表单主体隐藏；
- `configurable=false` 应与动作搭配，避免写入配置。

#### 文件选择动作

```json
{
  "file_picker": {
    "filters": [
      { "name": "配置文件", "extensions": ["json", "yaml"] }
    ]
  },
  "payload": { "path": "{{pickedFile}}" }
}
```

- 仅 Electron 环境可用；
- 用户取消时不发送请求；
- 首个文件路径写入动作上下文 `pickedFile`。

#### 会话动作

`session` 支持：

| 属性 | 默认值/作用 |
|---|---|
| `response_task_id_key` | `taskId`，从启动响应读取 WS topic |
| `stop_path` | 可选结束接口 |
| `stop_method` | POST |
| `stop_payload` | 支持模板 |
| `overlay_title` | 遮罩标题 |
| `overlay_description` | 遮罩说明 |
| `stop_label` | `结束会话` |
| `start_message` | 启动成功提示 |
| `success_message` | 完成提示 |
| `stop_message` | 主动结束提示 |
| `timeout_ms` | 超时时间；0/缺失不计时 |
| `timeout_auto_stop` | 超时后是否调用结束流程 |
| `timeout_message` | 超时提示 |
| `completion_type` | `Signal` |
| `completion_field` | `Accomplish` |
| `error_field` | `Error` |

会话订阅的是主 WebSocket topic，不是任意插件原生 WS URL。

### 5.2 自动完成文本

条件：字符串类型 + 非空 `options/enum` + `allow_custom: true`。

```json
{
  "type": "string",
  "label": "设备",
  "options": [
    { "label": "默认设备", "value": "default" },
    { "label": "模拟器 A", "value": "emu-a" }
  ],
  "allow_custom": true
}
```

- 可选择既有值，也可输入自定义字符串；
- 输入匹配 label 时保存对应 value；
- 输入匹配已有 value 时显示其 label；
- 自定义值在 blur 时去除首尾空白后保存；
- 仅字符串类型可触发。

### 5.3 有序多选

```json
{
  "type": "multiselect",
  "label": "任务顺序",
  "selection_mode": "ordered",
  "options": ["daily", "shop", "battle"]
}
```

- 显示带序号的开关按钮；
- 值必须为数组；
- 选择结果始终按 options 声明顺序规范化；
- 它不是拖拽排序，也不保存点击先后顺序；
- 不支持未知自定义项，旧值中不在 options 的项会被过滤。

### 5.4 多选

触发：`type: multiselect`，或 `type: list/list[...]` 且有 options/enum。

```json
{
  "type": "multiselect",
  "label": "启用功能",
  "options": [
    { "label": "功能 A", "value": "a" },
    { "label": "功能 B", "value": "b" }
  ]
}
```

- 使用 Select multiple；
- 值必须为数组；
- 选项 value 可为 string、number、boolean 等；
- list 一旦带选项就不再显示逐行列表编辑器。

### 5.5 单选

触发：`type: select`，或非 list/非 multiselect 字段带 options/enum。

```json
{
  "type": "select",
  "label": "运行模式",
  "default": "auto",
  "options": [
    { "label": "自动", "value": "auto" },
    { "label": "手动", "value": "manual" }
  ]
}
```

- options 优先于 enum；
- 简写值会归一成 `{label: String(value), value}`；
- `option_labels` 在后端生成阶段用于建立显示名；前端只消费最终 options；
- 任意普通字段只要带选项，通常都会转成单选，而不是原控件。

### 5.6 布尔开关

```json
{ "type": "boolean", "label": "启用", "default": true }
```

别名：`boolean`、`bool`。

- Switch 显示“是/否”；
- 使用 `Boolean(value)` 显示状态；字符串 `"false"` 仍会显示为真，后端必须提供真正 boolean。

### 5.7 路径选择

```json
{
  "type": "file",
  "label": "配置文件",
  "path_kind": "file",
  "filters": [
    { "name": "JSON", "extensions": ["json"] }
  ]
}
```

类型：`folder`、`file`、`path`。

- 值为字符串；
- `folder` 或 `path_kind=folder` 使用目录选择器；否则使用文件选择器；
- `filters` 仅传给文件选择器；
- Electron API 可用时显示“选择”按钮；浏览器环境只显示输入框；
- 前端不检查路径是否存在，后端 validator 负责真实性约束。

### 5.8 密码输入

正确声明：

```json
{
  "type": "string",
  "format": "password",
  "label": "访问令牌",
  "sensitive": true
}
```

- 只有“字符串类型 + `format=password`”进入密码控件；
- 当前 `type: password` 不属于字符串类型判断，会落入普通文本回退控件；不要使用；
- `sensitive=true` 本身不切换控件、不加密、不脱敏；
- 是否加密由后端配置 validator/存储层负责。

### 5.9 多行文本

```json
{
  "type": "string",
  "format": "textarea",
  "label": "备注",
  "rows": 6,
  "constraints": { "max_length": 2000 }
}
```

- 仅字符串类型 + `format=textarea`；
- `rows` 正数生效，默认 4；
- `max_length` 同时约束输入 maxlength 和保存校验。

### 5.10 普通文本

专属字符串别名：

- `string`
- `str`
- `uuid`
- `datetime`
- `related-id`
- `readonly`

```json
{
  "type": "string",
  "label": "名称",
  "placeholder": "请输入名称",
  "constraints": {
    "min_length": 1,
    "max_length": 40,
    "pattern": "^[A-Za-z0-9_-]+$"
  }
}
```

- `datetime` 当前是文本输入，不是日期选择器；其 format 不驱动前端日期格式；
- `uuid` 当前不做专属 UUID 前端校验；
- `related-id` 当前是文本输入，关联合法性由后端处理；
- `readonly` 是禁用文本输入；
- `format=url` 校验 http/https；
- `format=email` 当前没有专属控件或 email 校验；
- pattern 是 JavaScript 正则；非法正则会被前端忽略，后端仍应校验。

### 5.11 滑动条

```json
{
  "type": "slider",
  "label": "透明度",
  "default": 80,
  "min": 0,
  "max": 100,
  "step": 1
}
```

- 同时显示 Slider 和 InputNumber；
- 无值时 Slider 使用 min，仍无 min 时用 0；
- 范围读取优先级：顶层 min/max → constraints.ge/le → constraints.gt/lt；
- 前端把 gt/lt 当作可取边界，没有实现严格不等，后端必须再次校验；
- step 优先顶层 step，再取 `constraints.multiple_of`。

### 5.12 数字输入

别名：`number`、`integer`、`int`、`float`。

```json
{
  "type": "integer",
  "label": "重试次数",
  "min": 0,
  "max": 10,
  "step": 1
}
```

- integer/int 默认 step=1；number/float 默认由组件决定；
- 可用 min/max/step 或 constraints；
- 前端只验证有限数字和范围；整数性、multiple_of 严格性由后端保证。

### 5.13 列表编辑器

```json
{
  "type": "list[string]",
  "label": "启动参数",
  "item_type": "string",
  "default": []
}
```

- 支持 `type=list` 或 `list[...]`；
- `item_type` 优先于泛型中的类型；
- 专属行控件只识别 `string`、`number`、`boolean`；其他 item_type 也按字符串输入；
- number 新行默认 0，boolean 默认 false，其余默认空字符串；
- 仅支持增删行，不支持拖拽排序；数组顺序即显示顺序；
- 带 options/enum 时会变成多选控件。

### 5.14 键值表

```json
{
  "type": "key_value",
  "label": "环境变量",
  "default": { "LANG": "zh_CN.UTF-8" }
}
```

- 值必须是非数组对象；
- 新键自动为 `key_1`、`key_2`；
- key 在 blur 时更新；空 key 不生效；
- value 一律编辑和保存为字符串；
- 重命名为已存在 key 会覆盖已有值；当前没有重复键确认。

### 5.15 动态表格

```json
{
  "type": "table",
  "label": "映射表",
  "default": [
    { "source": "a", "target": "b" }
  ]
}
```

- 值必须是对象数组；
- 列名由所有现有行的 key 合并推断；
- 空表默认显示 `col_1`；
- 可新增行和新增列；新增列名自动为 `col_N`；
- 不支持重命名或删除列；
- 所有单元格都按字符串编辑；
- 不支持声明固定列 schema、列类型或逐列校验；复杂结构应使用 JSON 或 Custom Element。

### 5.16 JSON 编辑器

```json
{
  "type": "json",
  "label": "高级配置",
  "json_type": "object",
  "rows": 10,
  "default": {}
}
```

- 显示格式化 JSON 文本；
- blur 时解析并更新；空文本转为 `{}`；
- 解析失败显示“JSON 格式无效”并保留原模型；
- 当前前端保存校验要求 `typeof value === object`，数组也会通过该检查；但提示写作“JSON 对象”；
- 后端 DSL 支持 `json_type=object|array`，实际可否保存数组最终取决于后端 validator；
- 不提供 JSON Schema、属性级提示或代码补全。

### 5.17 回退文本输入

任何未命中前述条件的 type 都使用普通 Input，例如：

- `email`
- `password`（错误写法）
- `dict`、`dict[...]`
- `tuple[...]`
- `tag`（若强制显示）
- `multiple`（若强制进入 Schema）
- 自定义未知类型

回退控件会把显示值 `String(value)`，更新后保存字符串。不要把后端“能解析/能校验某类型”等同于前端“有对应编辑器”。

## 6. 选项系统

### 选项格式

```json
{
  "options": [
    "simple",
    2,
    true,
    { "label": "高级", "value": "advanced" }
  ]
}
```

简写项的 label 是 `String(value)`。

### 动态选项

`options_provider` 是后端元数据，前端不会主动调用 provider。后端必须在返回 Schema 前将其解析为最终 `options`。

```python
PluginField.select(
    "Device",
    "设备",
    "",
    [],
    options_provider={"provider": "device_options"},
)
```

注意：

- provider 不可用时要有明确空态或 allow_custom 策略；
- 动态 options 变化会影响有序多选归一化；
- 不应让前端依赖未解析的 options_provider 自行请求。

## 7. 布局尺寸

`SchemaForm` 默认 `layout=plugin-grid`，使用 12 列布局。

| `size` | 实际宽度 | 栅格跨度 |
|---|---:|---:|
| `1/4` | 25% | 3 |
| `1/3` | 33.3% | 4 |
| `1/2` | 50% | 6 |
| `2/3` | 66.7% | 8 |
| `3/4` | 75% | 9 |
| `1/1` | 100% | 12 |
| `small` | `1/3` | 4 |
| `half` | `1/2` | 6 |
| `medium` | `2/3` | 8 |
| `large` | `1/1` | 12 |
| 缺失/非法 | `1/3` | 4 |

窄屏下 Grid 会转为单列。`layout=single` 时 size 不影响宽度。

建议：

- boolean、短数字：`1/4` 或 `1/3`；
- 普通文本/select：`1/2`；
- 路径、textarea、JSON、list、table：`1/1`；
- action 按钮按实际位置决定，页头动作通常不需要 size。

## 8. 前端校验全集

| 元素 | 前端校验 |
|---|---|
| 全部非动作字段 | required：`undefined/null/空字符串` 视为空 |
| 字符串别名 | min_length、max_length、pattern、format=url |
| 数字/滑动条 | 有限数字、min/max |
| list/multiselect | 必须为数组 |
| json | 必须为 object；数组也属于 object |
| key_value | 必须是非数组对象 |
| table | 必须是数组 |
| 动作 | 跳过字段值校验 |
| 回退类型 | 除 required 外无专属校验 |

前端校验只用于即时反馈，不能替代后端 Pydantic、ConfigBase validator 或 API 边界校验。

## 9. 后端 DSL 快捷元素

`PluginField` 当前直接提供：

| DSL | 最终类型 | UI 行为 |
|---|---|---|
| `PluginField.string()` | string | 文本，可配 textarea/password/options |
| `PluginField.boolean()` | boolean | 开关 |
| `PluginField.select()` | select | 单选 |
| `PluginField.number()` | number | 数字，可用 ui_type=slider 变滑动条 |
| `PluginField.file()` | file | 文件路径 |
| `PluginField.folder()` | folder | 文件夹路径 |
| `PluginField.json()` | json | JSON 文本区 |
| `PluginField.datetime()` | datetime | 当前为普通文本 |
| `PluginField.related_id()` | related-id | 当前为普通文本，后端关联校验 |
| `PluginField.virtual()` | readonly | 只读计算文本 |
| `PluginField.tag()` | tag | 默认 hidden，无专属控件 |
| `PluginField.button()` | button | 动作，不持久化 |
| `PluginField.multiple()` | multiple | 嵌套 ConfigBase，默认不进 Schema |
| `PluginField.group()` | 非字段 | 构建分组 |

`tag` 和 `multiple` 是后端结构能力，不应被当成成熟的前端可视元素。

## 10. 已知差异与使用禁区

1. `type=password` 不会显示密码框；必须用字符串类型 + `format=password`。
2. `sensitive=true` 不等于密码框、加密或响应脱敏。
3. `datetime` 没有日期选择器，`email` 没有专属输入与校验。
4. `ui_type` 不由前端直接读取；最终必须生成正确 `type`。
5. `configurable=false` 不会自动隐藏字段；需要 `hidden/include_in_schema` 或动作字段语义配合。
6. `icon` 只对页头动作有可见用途，普通字段不会显示图标。
7. `ordered` 多选按 options 顺序，不按用户点击顺序。
8. `table` 和 `key_value` 的值均被字符串化，不适合强类型复杂数据。
9. `dict/tuple/Optional/Union/Literal` 是后端类型表达能力，不都对应专属 UI。Literal 通常应生成 options/select。
10. `multiple` 默认 `include_in_schema=false`，当前没有嵌套子表单渲染器。
11. 前端只消费最终 options；`options_provider` 必须在后端解析。
12. 任何未知 type 都会静默回退文本框，审查时不能只看“页面能显示”。

## 11. 元素选择决策

- 真/假：boolean；
- 有限单值：select；
- 有限多值：multiselect；
- 固定声明顺序的多选：ordered multiselect；
- 推荐值但允许自定义：string + options + allow_custom；
- 普通自由文本：string；
- 长文本：string + format=textarea；
- 密钥：string + format=password + sensitive + 后端加密；
- 数字：number/int/float；
- 范围型数字：slider；
- 本地文件/目录：file/folder；
- 同类型简单数组：list；
- 字符串键值：key_value；
- 简单二维字符串表：table；
- 未知或复杂 JSON：json；
- 触发后端行为：button/action；
- 日期、复杂表格、树、代码编辑器、富交互或实时视图：当前 SchemaForm 不足，使用 Custom Element。

## 12. 完整字段示例

```json
{
  "groups": [
    {
      "key": "Info",
      "label": "基本信息",
      "fields": [
        {
          "key": "Info.Name",
          "type": "string",
          "label": "名称",
          "required": true,
          "placeholder": "请输入名称",
          "constraints": { "min_length": 1, "max_length": 40 },
          "size": "1/2"
        },
        {
          "key": "Info.Enabled",
          "type": "boolean",
          "label": "启用",
          "default": true,
          "size": "1/4"
        },
        {
          "key": "Info.Mode",
          "type": "select",
          "label": "模式",
          "options": [
            { "label": "自动", "value": "auto" },
            { "label": "手动", "value": "manual" }
          ],
          "size": "1/4"
        },
        {
          "key": "Info.RootPath",
          "type": "folder",
          "label": "项目目录",
          "required": true,
          "size": "1/1"
        }
      ]
    },
    {
      "key": "Action",
      "label": "动作",
      "fields": [
        {
          "key": "Action.Check",
          "type": "button",
          "label": "检查项目",
          "icon": "check",
          "configurable": false,
          "action": {
            "label": "检查项目",
            "path": "/plugin/example/check",
            "method": "POST",
            "payload": {
              "rootPath": "{{formModel.Info.RootPath}}"
            },
            "refresh": true
          }
        }
      ]
    }
  ]
}
```

## 13. 元素审查清单

- [ ] 最终 Schema 的 type 能命中预期渲染分支；
- [ ] 值类型与控件一致，没有字符串布尔/数字；
- [ ] 带 options 的字段不会意外覆盖原控件；
- [ ] password 使用 string + format=password；
- [ ] sensitive 字段有后端加密和响应保护；
- [ ] list 的 item_type 仅依赖当前支持的 string/number/boolean；
- [ ] table/key_value 只承载字符串型简单结构；
- [ ] 动作字段 configurable=false，path、method、payload 明确；
- [ ] 会话动作的 task ID、完成消息、停止接口和超时行为一致；
- [ ] options_provider 已在后端解析为 options；
- [ ] readonly/hidden/configurable/include_in_schema 没有混淆；
- [ ] size 与控件复杂度匹配；
- [ ] 前端和后端都验证关键约束；
- [ ] 未知类型没有因回退文本框而掩盖实现缺失；
- [ ] 超出 SchemaForm 能力的交互已改用 Custom Element。
