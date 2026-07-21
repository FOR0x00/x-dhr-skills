---
name: x-dhr
description: X-DHR 智能人力资源助手 - 检索个人/员工/花名册/人事信息；发起审批、请假、出差、加班、报销、计划、申请等流程和表单
version: 1.0.0
---

# X-DHR 智能人力资源助手

用于执行 `X-DHR` 数据查询和业务操作，支持查询员工/花名册信息、发起审批流程和填写表单等能力。

## 支持的能力

| 能力                         | 用户示例                                     | 详细说明                                  |
|------------------------------|----------------------------------------------|-------------------------------------------|
| 检索员工、花名册等人事信息时 | 帮我查下张三的人事信息、查询我的人事信息     | [org-person.md](references/org-person.md) |
| 发起审批流程、填写表单       | 帮我发起请假单、出差、加班、报销、计划、申请 | [workflow.md](references/workflow.md)     |

## 工具调用规则

- 使用 `curl` 调用 MCP HTTP JSON-RPC 接口
- 工具调用依赖环境变量 `X_DHR_MCP_URL` 和 `X_DHR_MCP_TOKEN`。如果仍无法解析到，禁止执行工具，提示用户配置后重试。
- 优先使用环境变量。只有用户明确要求长期保存配置时，才可以写入
  `~/.x-dhr/config.json`，并必须设置为仅当前用户可读写权限，例如 `chmod 600 ~/.x-dhr/config.json`。
- 调用某工具时不要尝试猜测工具的参数，一定先调用 `tools/list` 查看工具的定义信息；同一会话已查看过并缓存目标工具
  `inputSchema` 后不要重复调用 `tools/list`。
- `curl` 调用遵循 MCP tools 语义：`tools/list` 仅在首次调用工具或不确定参数时使用；`tools/call` 传入实际工具名和工具定义要求的
  `arguments`。
- 工具调用过程仅内部执行，不展示工具名、调用标题、内部字段 ID、查询表达式、参数、命令、原始 JSON、原始返回或 token。
- 写操作：必须先得到用户明确确认，且除非用户要求，不要自动重试。
- 同一轮或同一会话内优先复用已获取的 Schema 和模板定义。
- 只有出现以下情况时才刷新对应缓存：工具明确返回表单 Schema 变化、模板失效、权限变化、用户明确切换流程目标、用户要求重新加载最新数据。
- 缓存只用于减少重复读取 Schema 和模板定义；不要缓存或复用会变化的业务查询结果、审批提交结果。
- 错误处理：工具返回 `code != 0` 时停止本次工具调用，并用可读方式向用户说明
  `message` 或错误原因。除非用户明确要求，不要自动重试；不要建议绕过权限或更换未授权接口。

`curl` 调用时先组装环境变量：

```bash
X_DHR_MCP_URL="${X_DHR_MCP_URL:-}"
X_DHR_MCP_TOKEN="${X_DHR_MCP_TOKEN:-}"
if { [ -z "$X_DHR_MCP_URL" ] || [ -z "$X_DHR_MCP_TOKEN" ]; } && [ -f ~/.x-dhr/config.json ]; then
  X_DHR_MCP_URL=$(jq -r '.X_DHR_MCP_URL' ~/.x-dhr/config.json)
  X_DHR_MCP_TOKEN=$(jq -r '.X_DHR_MCP_TOKEN' ~/.x-dhr/config.json)
fi
```

`curl` 调用的 JSON-RPC 示例：

```bash
curl -s "$X_DHR_MCP_URL" \
  -H "Authorization: Bearer ${X_DHR_MCP_TOKEN}" \
  -H "Content-Type: application/json; charset=utf-8" \
  -H "Accept: application/json, text/event-stream" \
  --data '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}'
```

```bash
curl -s "$X_DHR_MCP_URL" \
  -H "Authorization: Bearer ${X_DHR_MCP_TOKEN}" \
  -H "Content-Type: application/json; charset=utf-8" \
  -H "Accept: application/json, text/event-stream" \
  --data '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"TOOL_NAME","arguments":{}}}'
```

## 通用对象 Schema

`X-DHR` 的大多数业务都支持自定义字段和明细表，开展业务前通常需要获取对应表单定义。

- `McpForm`：`formId`、`formName`、`fields`、可选 `detailForms`。
- `McpField`：`fieldId`、`fieldName`、`fieldType`、`showType`、`queryExpression`、`options`、`validate`。
- `queryExpression`：字段支持的查询操作符，可能是逗号分隔的多个操作符；组装查询条件时只能选择其中一个。
- `validate`：可能包含 `required`、`minLength`、`maxLength`、`minValue`、`maxValue`。
- `McpData`： `formId`、`fieldDataList[{fieldId, fieldValue}]`、可选 `detailDataList`。

## 条件查询规则

- `McpCondition`：`field`、`expression`、`value`
- `field` 必须使用当前花名册 `Schema` 中的 `McpField.fieldId`；`expression` 必须从该字段
  `McpField.queryExpression` 支持的操作符中选择一个。
- 当用户输入并且/或者条件时，组合条件本身的 `expression` 可为 `and`/`or`，`value` 为
  `list[McpCondition]`；组合条件的 `field` 为空字符串。

查询条件 JSON 示例。示例中的 `field` 是占位符，必须替换为本次获取的 `Schema` 中对应字段的真实
`fieldId`；不要把示例字段名当作固定 ID 使用：

```json
[
  {
    "field": "<从当前 Schema 中选择的姓名字段 fieldId>",
    "expression": "like",
    "value": "张三"
  }
]
```

```json
[
  {
    "field": "",
    "expression": "or",
    "value": [
      {
        "field": "<从当前 Schema 中选择的姓名字段 fieldId>",
        "expression": "like",
        "value": "张三"
      },
      {
        "field": "<从当前 Schema 中选择的姓名字段 fieldId>",
        "expression": "like",
        "value": "李四"
      }
    ]
  }
]
```

## 字段类型规则

- `textfield`, `textarea`：字符串，保留用户原意，执行长度、正则等校验。
- `cellphone`：手机号字符串，转换为手机号。
- `number`, `numberonline`, `score`：数字，去除合法千分位，检查精度、最小值和最大值；不要把模糊数量词转数字。
- `checkbox`：布尔值，把是否、开关等语义转化为 `true`/`false`。
- `datepicker`：结合当前日期和时区解析相对表达；`showType=0` 转为 `yyyy-MM-dd`，`showType=1` 转为 `yyyy-MM-dd HH:mm`，
  `showType=4` 转为 `yyyy-MM`。
- `timepicker`：转换为 `HH:mm`。
- `radio`, `singleselect`：单个值，必须来自 `options`。
- `multiselect`：多个值，每项都必须来自 `options`；多项时按工具要求用英文 `,` 分割。
- `selectperson`, `multiselectperson`：提交人员名称；多项时英文 `,` 分割。
- `selectdept`, `multiselectdept`：提交部门名称；多项时英文 `,` 分割。
- `selectjob`, `selectjoblevel`, `selectcompany`, `selectmultiorg`：提交显示名称；多项时英文 `,` 分割。
- `address`, `city`, `lbs`, `attachment`, `image`, `imagesingle`, `imagemulti`, `signature`, `outercontrol`：不支持通过
  Agent 填写。

日期时间额外规则：

- `今天`、`明天`、`下周一` 等相对日期可以按当前时区换算，但要向用户展示换算后的绝对日期。
- `showType=1` 时，`上午`、`下午`、`晚上`、`下班前` 等时段词不等于精确时刻。不要自行采用 09:00、12:00、18:00 等默认时刻，必须追问具体时间。
- 若字段类型不支持且是必填字段，告知用户该业务不支持通过 Agent 完成，请登录系统手动操作。
- 未知 `fieldType` 不要降级为普通文本。使用 `AskUserQuestion` 获取用户输入后仍保留原类型。

## 字段值匹配规则

只从当前消息和明确的对话上下文中提取字段值，包括显式的“字段=值”、自然语言中的时间、时长、人员、部门、原因、金额，以及用户已经确认过的值。不要把假设、常识默认值或模型推断当作用户已提供的数据。

匹配顺序：

1. 字段名称、字段标题或用户显式键完全匹配。
2. 字段名称命中当前业务模块定义的字段别名或业务映射，并且只命中一个候选字段。
3. 唯一字段与用户表达的业务含义明确匹配，例如“请假原因”匹配原因字段。
4. 值的类型、格式和可选项与字段约束兼容。

遇到以下任一情况时不要自动匹配：

- 用户值不满足类型、范围、长度、格式或枚举约束。
- 字段依赖另一个尚未确定的字段。
- 别名或业务映射命中多个候选字段，无法唯一确定。
- 只有弱语义推断，没有字段标题、别名或业务上下文支持。

## AskUserQuestion 规则

- 一次最多询问 3 个短问题；字段较多时分轮进行。
- 问题标题（`header`）使用字段标题，问题正文（`question`）包含期望类型、格式、范围或示例。
- 枚举和消歧场景直接提供候选选项（`options`，含 `label` 和 `description`）；推荐项只在有明确系统依据时标记。
- 用户回答后立即做类型校验。无效时说明具体约束并再次询问，不要静默转换成其他值。
- 用户拒绝提供必填字段时，停止准备并说明无法继续的字段。

如果宿主确实没有 `AskUserQuestion` 或等效的交互式提问能力，才用简短普通问题作为降级方案；降级时一次仍最多询问 3 个问题。
