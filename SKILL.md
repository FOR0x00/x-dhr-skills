---
name: x-dhr
description: X-DHR 智能人力资源助手 - 检索企业员工信息、花名册；发起审批、表单、流程
version: 1.0.0
---

# X-DHR 智能人力资源助手

## 前置条件

- 优先使用名为 `x-dhr` 的 MCP server；如果该 MCP server 可用，不要求在当前 shell 中解析到任何环境变量。
- 仅当没有可用的 `x-dhr` MCP server、需要降级使用 `curl` 调用 MCP HTTP 接口时，才要求解析到 `X_DHR_MCP_URL` 和
  `X_DHR_MCP_TOKEN`。如果仍无法解析到，禁止执行工具，提示用户配置后重试。
- 不要在对话中请求、展示或回显 `X_DHR_MCP_TOKEN`。如果用户主动提供 token，只能用于本次必要调用，不要复述。

**MCP 配置:**

```json
{
  "x-dhr": {
    "type": "streamableHttp",
    "url": "${X_DHR_MCP_URL}",
    "headers": {
      "Authorization": "Bearer ${X_DHR_MCP_TOKEN}"
    }
  }
}
```

## 支持的能力

| 能力                         | 用户示例                         | 详细说明                                  |
|------------------------------|----------------------------------|-------------------------------------------|
| 检索员工、花名册等人事信息时 | 帮我查下张三的人事信息           | [org-person.md](references/org-person.md) |
| 发起审批流程、填写表单       | 帮我发起请假单、出差、加班、报销 | [workflow.md](references/workflow.md)     |

## 通用规则

### 通用对象 Schema

`X-DHR` 的大多数业务都可以自定义字段，可自定义多个明细表，开展业务前通常都需要获取该业务的表单定义，封装成了以下对象结构如下：

**R Mcp工具调用结果Schema**

- code：状态码，0代表成功；非0表示异常，`message` 为异常信息
- message：异常信息
- data: 工具调用结果数据，可能为对象，可能为 `list`

**McpForm 表单 Schema**

- formId: 主表单/明细表单ID
- formName: 表单显示名称
- fields[McpField]: 字段列表
- detailForms[McpForm]: 明细表单列表，可选

**McpField 字段 Schema**

- fieldId: 字段ID
- fieldName: 字段显示名称
- fieldType: 数据类型或控件类型，参考字段类型清单说明
- showType: 字段的显示模式，不同模式处理规则可能不同
- queryExpression: 组装查询条件时字段支持的表达式，用`,`分隔
- options: 枚举类型、下拉类型、单选等选项列表
- validate.required: 字段是否必填
- validate.minLength: 文本类型时，最小文本长度
- validate.maxLength: 文本类型时，最大文本长度
- validate.minValue: 数字类型时，最小数值
- validate.maxValue: 数字类型时，最大数值

**字段类型清单**

| fieldType                                                        | 字段类型                     | 处理规则                                                                                                                                                    |
|------------------------------------------------------------------|------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `textfield`, `textarea`                                          | 字符串                       | 保留用户原意；执行长度、正则等校验                                                                                                                          |
| `cellphone`                                                      | 手机号字符串                 | 转换为手机号                                                                                                                                                |
| `number`, `numberonline`, `score`                                | 数字                         | 去除合法千分位；检查精度、最小值和最大值。不要把模糊数量词转数字。                                                                                          |
| `checkbox`                                                       | 开关                         | 布尔值；把是否，开关等语义转化为 true/false                                                                                                                 |
| `datepicker`                                                     | 日期                         | 结合当前日期和时区解析相对表达；`showType=0`转换为yyyy-MM-dd，`showType=1`转换为yyyy-MM-dd HH:mm，`showType=4`转换为yyyy-MM；日期或精度有歧义时使用 UserAsk |
| `timepicker`                                                     | 时间                         | 结合当前日期和时区解析相对表达；转换为HH:mm格式，有歧义时使用 UserAsk                                                                                       |
| `radio`, `singleselect`                                          | 单个枚举项                   | 单个值；这个值必须来自 `options`                                                                                                                            |
| `multiselect`                                                    | 枚举数组                     | 可多个值；每项都必须来自 `options`；多项时英文 `,` 分割                                                                                                     |
| `selectperson`, `multiselectperson`                              | 人员或多个人员               | 提交人员名称；多项时英文`,`分割                                                                                                                             |
| `selectdept`, `multiselectdept`                                  | 部门或多个部门               | 提交部门名称；多项时英文`,`分割                                                                                                                             |
| `selectjob`, `selectjoblevel`, `selectcompany`, `selectmultiorg` | 职位、职位等级、公司、多组织 | 提交显示名称；多项时英文 `,` 分割                                                                                                                           |
| `address`, `city`, `lbs`                                         | 地理位置                     | 提示用户不支持该控件、字段                                                                                                                                  |
| `attachment`, `image`, `imagesingle`, `imagemulti`, `signature`  | 文件或手写签名控件           | 提示用户不支持该控件、字段                                                                                                                                  |
| `outercontrol`                                                   | 加载外部数据控件             | 提示用户不支持该控件、字段                                                                                                                                  |

未知 `fieldType` 不要降级为普通文本。使用 UserAsk 获取用户输入后仍保留原类型

**补充说明**

1. `datepicker`, `timepicker` 日期时间额外遵循以下规则：

- `今天`、`明天`、`下周一`等相对日期可以按当前时区换算，但要向用户展示换算后的绝对日期。
- `showType=1` 时，`上午`、`下午`、`晚上`、`下班前`等时段词不等于精确时刻。不要自行采用 09:00、12:00、18:00 等默认时刻。必须使用
  UserAsk 询问具体时间。
- 若有字段类型不支持，但又是必填的情况，则告知用户该业务不支持通过 `Agent` 完成，请登录系统手动操作

**McpData 表单提交数据 Schema**

- formId: 表单ID
- fieldDataList[McpFieldData]: 字段数据列表
- detailDataList[McpData]: 明细表单数据列表

**McpFieldData 表单提交字段数据 Schema**

- fieldId: 字段ID
- fieldValue: 字段值

### McpCondition 查询条件 Schema

- field: 字段ID
- expression: 查询表达式，从 `McpField` 的 `queryExpression` 中取值
- value: 查询值

**规则：**

- 当用户输入的查询条件为并且时 `expression` 值为 `and`，`value` 为 list[McpCondition]
- 当用户输入的查询条件为或者时 `expression` 值为 `or`，`value` 为 list[McpCondition]

### 字段值匹配规则

#### 1. 从当前消息和明确的对话上下文中提取：

- 字段线索：显式的“字段=值”、自然语言中的时间、时长、人员、部门、原因、金额等。
- 用户已经确认过的值。

#### 2. 不要把假设、常识默认值或模型推断当作用户已提供的数据。

#### 3. 按以下顺序匹配，每次匹配都必须同时满足字段语义与类型约束：

1. 字段名称或标题或与用户显式键完全匹配。
2. 唯一字段与用户表达的业务含义明确匹配，例如“请假原因”匹配原因字段。
3. 值的类型、格式和可选项与字段约束兼容。

#### 4. 遇到以下任一情况时不要自动匹配：

- 用户值不满足类型、范围、长度、格式或枚举约束。
- 字段依赖另一个尚未确定的字段。
- 只有弱语义推断，没有字段标题、别名或业务上下文支持。

#### 5. 优先询问必填字段。可选字段默认不追问，除非用户要求完整填写或该字段影响其他字段。

### UserAsk 遵循以下规则

- 一次最多询问 3 个短问题；字段较多时分轮进行。
- 问题标题使用字段标题，问题正文包含期望类型、格式、范围或示例。
- 枚举和消歧场景直接提供候选选项；推荐项只在有明确系统依据时标记。
- 用户回答后立即做类型校验。无效时说明具体约束并再次询问，不要静默转换成其他值。
- 用户拒绝提供必填字段时，停止准备并说明无法继续的字段。

如果宿主没有 `UserAsk` 能力，用简短普通问题作为降级方案；一次仍最多询问 3 个字段。

### 会话级缓存规则

为减少重复工具调用，同一轮或同一会话内必须优先复用已经获取过的工具定义和业务 Schema。

- 只有出现以下情况时才刷新对应缓存：工具明确返回表单 Schema 变化、流程模板失效、权限变化、用户明确切换流程目标、用户要求重新加载最新数据。
- 缓存只用于减少重复读取 Schema 和模板定义；不要缓存或复用会变化的业务查询结果、审批提交结果。

## MCP curl 调用模式

### 规则

- 当通过 `curl` 调用 MCP HTTP 接口时，优先使用环境变量。只有用户明确要求长期保存配置时，才可以写入 `~/.x-dhr/config.json`
  ，并必须设置为仅当前用户可读写权限，例如 `chmod 600 ~/.x-dhr/config.json`。
- 首次调用某工具前，或不确定参数时，先查看工具的定义信息；同一会话已查看过工具定义后不要重复调用 `tools/list`
  。已知工具名和参数稳定时，直接调用目标工具。
- 工具调用过程仅内部执行，不展示任何工具调用痕迹：禁止向用户展示 `X_DHR_MCP_TOKEN` 、工具名、调用标题、命令、参数、原始返回；只给用户可理解的结果
- 向用户展示摘要时，只展示字段标题和显示值；不要展示字段ID、查询表达式、工具参数或原始 JSON。

建议先组装环境变量：

```bash
X_DHR_MCP_URL="${X_DHR_MCP_URL:-}"
X_DHR_MCP_TOKEN="${X_DHR_MCP_TOKEN:-}"
if { [ -z "$X_DHR_MCP_URL" ] || [ -z "$X_DHR_MCP_TOKEN" ]; } && [ -f ~/.x-dhr/config.json ]; then
  X_DHR_MCP_URL=$(jq -r '.X_DHR_MCP_URL' ~/.x-dhr/config.json)
  X_DHR_MCP_TOKEN=$(jq -r '.X_DHR_MCP_TOKEN' ~/.x-dhr/config.json)
fi
```

### 查看工具列表及定义

```bash
curl -s "${X_DHR_MCP_URL}" \
  -H "Authorization: Bearer ${X_DHR_MCP_TOKEN}" \
  -H "Content-Type: application/json; charset=utf-8" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}'
```

### 调用工具：

若 `arguments` 请求参数包含中文等非 ASCII 字符时，不要把 JSON 直接写在 Windows `cmd` 或 PowerShell 命令行参数中；必须先生成
UTF-8 编码的 payload 文件，再用 `curl.exe --data-binary "@payload.json"` 发送。

```bash
curl -s "${X_DHR_MCP_URL}" \
  -H "Authorization: Bearer ${X_DHR_MCP_TOKEN}" \
  -H "Content-Type: application/json; charset=utf-8" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"TOOL_NAME","arguments":{}}}'
```

将 `TOOL_NAME` 替换为实际工具名，将 `arguments` 替换为工具定义要求的参数。从 `result.content[0].text`、
`result.structuredContent` 解析返回结果。
