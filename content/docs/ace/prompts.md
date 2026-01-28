---
summary: 提示是用于用户输入的终端小部件，使用 @poppinss/prompts 包。它们支持输入、密码和选择等多种类型，并为测试集成而设计。
---

# 提示

提示是您可用于接受用户输入的交互式终端小部件。Ace 提示由 [@poppinss/prompts](https://github.com/poppinss/prompts) 包提供支持，该包支持以下提示类型。

- input
- list
- password
- confirm
- toggle
- select
- multi-select
- autocomplete

Ace 提示的构建考虑了测试。编写测试时，您可以捕获提示并以编程方式响应它们。

另请参阅：[测试 ace 命令](../testing/console_tests.md)

## 显示提示

您可以使用 `this.prompt` 属性在所有 Ace 命令上显示提示。

```ts
import { BaseCommand } from '@adonisjs/core/ace'

export default class GreetCommand extends BaseCommand {
  async run() {
    const modelName = await this.prompt.ask('输入模型名称')

    console.log(modelName)
  }
}
```

## 提示选项

以下是提示接受的选项列表。您可以参考此表作为单一事实来源。

<table>
<tr>
<td width="110px">选项</td>
<td width="120px">接受者</td>
<td>描述</td>
</tr>
<tr>
<td>

`default` (字符串)

</td>

<td>

所有提示

</td>

<td>

未输入值时使用的默认值。对于 `select`、`multiselect` 和 `autocomplete` 提示，该值必须是选项数组索引。

</td>
</tr>

<tr>
<td>

`name` (字符串)

</td>

<td>

所有提示

</td>

<td>

提示的唯一名称

</td>
</tr>

<tr>
<td>

`hint` (字符串)

</td>

<td>

所有提示

</td>

<td>

显示在提示旁边的提示文本

</td>
</tr>
<tr>
<td>

`result` (函数)

</td>

<td>所有提示</td>
<td>

转换提示返回值。`result` 方法的输入值取决于提示。例如，`multiselect` 提示值将是所选选项的数组。

```ts
{
  result(value) {
    return value.toUpperCase()
  }
}
```

</td>
</tr>

<tr>
<td>

`format` (函数)

</td>

<td>所有提示</td>

<td>

在用户键入时实时格式化输入值。格式化仅应用于 CLI 输出，而不是返回值。

```ts
{
  format(value) {
    return value.toUpperCase()
  }
}
```

</td>
</tr>

<tr>
<td>

`validate` (函数)

</td>

<td>所有提示</td>

<td>

验证用户输入。从方法返回 `true` 将通过验证。返回 `false` 或错误消息字符串将被视为失败。

```ts
{
  validate(value) {
    return value.length > 6
    ? true
    : '模型名称必须至少 6 个字符长'
  }
}
```

</tr>

<tr>
<td>

`limit` (数字)

</td>

<td>

`autocomplete`

</td>

<td>

限制显示的选项数量。您必须滚动才能查看其余选项。

</td>
</tr>
</table>

## 文本输入

您可以使用 `prompt.ask` 方法呈现提示以接受文本输入。该方法接受提示消息作为第一个参数和[选项对象](#prompt-options)作为第二个参数。

```ts
await this.prompt.ask('输入模型名称')
```

```ts
// 验证输入
await this.prompt.ask('输入模型名称', {
  validate(value) {
    return value.length > 0
  },
})
```

```ts
// 默认值
await this.prompt.ask('输入模型名称', {
  default: 'User',
})
```

## 掩码输入

顾名思义，掩码输入提示在终端中掩码用户输入。您可以使用 `prompt.secure` 方法显示掩码提示。

该方法接受提示消息作为第一个参数和[选项对象](#prompt-options)作为第二个参数。

```ts
await this.prompt.secure('输入账户密码')
```

```ts
await this.prompt.secure('输入账户密码', {
  validate(value) {
    return value.length < 6 ? '密码必须至少 6 个字符长' : true
  },
})
```

## 选项列表

您可以使用 `prompt.choice` 方法显示单个选择的选项列表。该方法接受以下参数。

1. 提示消息。
2. 选项数组。
3. 可选[选项对象](#prompt-options)。

```ts
await this.prompt.choice('选择包管理器', ['npm', 'yarn', 'pnpm'])
```

要提及不同的显示值，您可以将选项定义为对象。`name` 属性作为提示结果返回，`message` 属性显示在终端中。

```ts
await this.prompt.choice('选择数据库驱动程序', [
  {
    name: 'sqlite',
    message: 'SQLite',
  },
  {
    name: 'mysql',
    message: 'MySQL',
  },
  {
    name: 'pg',
    message: 'PostgreSQL',
  },
])
```

## 多选选项

您可以使用 `prompt.multiple` 方法在选项列表中允许多个选择。接受的参数与 `choice` 提示相同。

```ts
await this.prompt.multiple('选择数据库驱动程序', [
  {
    name: 'sqlite',
    message: 'SQLite',
  },
  {
    name: 'mysql',
    message: 'MySQL',
  },
  {
    name: 'pg',
    message: 'PostgreSQL',
  },
])
```

## 确认操作

您可以使用 `prompt.confirm` 方法显示带有是/否选项的确认提示。该方法接受提示消息作为第一个参数和[选项对象](#prompt-options)作为第二个参数。

`confirm` 提示返回布尔值。

```ts
const deleteFiles = await this.prompt.confirm('想要删除所有文件吗？')

if (deleteFiles) {
}
```

要自定义是/否选项显示值，您可以使用 `prompt.toggle` 方法。

```ts
const deleteFiles = await this.prompt.toggle('想要删除所有文件吗？', [
  '是',
  '否',
])

if (deleteFiles) {
}
```

## 自动完成

`autocomplete` 提示是选择和多重选择提示的组合，但具有模糊搜索选项的能力。

```ts
const selectedCity = await this.prompt.autocomplete(
  '选择您的城市',
  await getCitiesList(),
)
```
