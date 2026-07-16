---
name: x-dhr
description: X-DHR 智能人力资源助手 - 检索企业员工信息、花名册；发起审批、表单、流程
version: 1.0.0
---

# X-DHR 智能人力资源助手

## 前置条件

- 必要环境变量: `X_DHR_MCP_URL` 和 `X_DHR_MCP_TOKEN`，如果没有解析到必要环境变量时禁止执行工具，提示用户配置或提供后重试
- 优先使用名为 `x-dhr` 的 MCP server，否则降级使用 `curl` 调用 MCP HTTP 接口

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

| 能力                 | 用户示例             | 详细说明                                      |
|--------------------|------------------|-------------------------------------------|
| 检索处理员工、花名册等智能人事事项时 | 帮我查下张三的人事信息      | [org-person.md](references/org-person.md) |
| 发起审批流程、填写表单        | 帮我发起请假单、出差、加班、报销 | [workflow.md](references/workflow.md)     |

## 通用规则

### 表单及表单数据Schema

`X-DHR` 的大多数业务都可以自定义字段，可自定义多个明细表，开展业务前都需要获取该业务的表单定义，封装成了以下对象结构如下：

**McpForm Schema**

- formId: 主表单/明细表单ID
- formName: 表单显示名称
- fields[McpField]: 字段列表
- detailForms[McpForm]: 明细表单列表，可选

**McpField Schema**

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

| fieldType                                                        | 字段类型           | 处理规则                                                                                                                 |
|------------------------------------------------------------------|----------------|----------------------------------------------------------------------------------------------------------------------|
| `textfield`, `textarea`                                          | 字符串            | 保留用户原意；执行长度、正则等校验                                                                                                    |
| `cellphone`                                                      | 手机号字符串         | 转换为手机号                                                                                                               |
| `number`, `numberonline`, `score`                                | 数字             | 去除合法千分位；检查精度、最小值和最大值。不要把模糊数量词转数字。                                                                                    |
| `checkbox`                                                       | 开关             | 布尔值；把是否，开关等语义转化为 true/false                                                                                          |
| `datepicker`                                                     | 日期             | 结合当前日期和时区解析相对表达；`showType=0`转换为yyyy-MM-dd，`showType=1`转换为yyyy-MM-dd HH:mm，`showType=4`转换为yyyy-MM；日期或精度有歧义时使用 UserAsk |
| `timepicker`                                                     | 时间             | 结合当前日期和时区解析相对表达；转换为HH:mm格式，有歧义时使用 UserAsk                                                                            |
| `radio`, `singleselect`                                          | 单个枚举项          | 单个值；这个值必须来自 `options`                                                                                                |
| `multiselect`                                                    | 枚举数组           | 可多个值；每项都必须来自 `options`；多项时英文 `,` 分割                                                                                  |
| `selectperson`, `multiselectperson`                              | 人员或多个人员        | 提交人员名称；多项时英文`,`分割                                                                                                    |
| `selectdept`, `multiselectdept`                                  | 部门或多个部门        | 提交部门名称；多项时英文`,`分割                                                                                                    |
| `selectjob`, `selectjoblevel`, `selectcompany`, `selectmultiorg` | 职位、职位等级、公司、多组织 | 提交显示名称；多项时英文 `,` 分割                                                                                                  |
| `address`, `city`, `lbs`                                         | 地理位置           | 提示用户不支持该控件、字段                                                                                                        |
| `attachment`, `image`, `imagesingle`, `imagemulti`, `signature`  | 文件或手写签名控件      | 提示用户不支持该控件、字段                                                                                                        |
| `outercontrol`                                                   | 加载外部数据控件       | 提示用户不支持该控件、字段                                                                                                        |

未知 `fieldType` 不要降级为普通文本。使用 UserAsk 获取用户输入后仍保留原类型

**补充说明**

1. `datepicker`, `timepicker` 日期时间额外遵循以下规则：

- `今天`、`明天`、`下周一`等相对日期可以按当前时区换算，但要向用户展示换算后的绝对日期。
- `showType=1` 时，`上午`、`下午`、`晚上`、`下班前`等时段词不等于精确时刻。不要自行采用 09:00、12:00、18:00 等默认时刻。必须使用
  UserAsk 询问具体时间。
- 若有字段类型不支持，但又是必填的情况，则告知用户该业务不支持通过 `Agent` 完成，请登录系统手动操作

**McpData Schema**

- formId: 表单ID
- fieldDataList[McpFieldData]: 字段数据列表
- detailDataList[McpData]: 明细表单数据列表

**McpFieldData Schema**

- fieldId: 字段ID
- fieldValue: 字段值

### 查询条件 Schema

- field: 字段ID
- expression: 查询表达式，从 `McpField` 的 `queryExpression` 中取值
- value: 查询值

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

## MCP curl 调用模式

### 规则

- 当通过 `curl` 调用 MCP HTTP 接口时，尝试询问用户是否记录到 `~/.x-dhr/config.json` 供后续对话复用
- 首次调用某工具前，或不确定参数时，先查看工具的定义信息
- 工具调用过程仅内部执行，不展示任何工具调用痕迹：禁止向用户展示 `X_DHR_MCP_TOKEN` 、工具名、调用标题、命令、参数、原始返回；只给用户可理解的结果

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
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}'
```

### 调用工具：

```bash
curl -s "${X_DHR_MCP_URL}" \
  -H "Authorization: Bearer ${X_DHR_MCP_TOKEN}" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"TOOL_NAME","arguments":{}}}'
```

将 `TOOL_NAME` 替换为实际工具名，将 `arguments` 替换为工具定义要求的参数。从 `result.content[0].text`、
`result.structuredContent` 解析返回结果。
