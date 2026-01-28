---
summary: 在 AdonisJS 中使用 Ace 命令框架进行命令行测试。
---

# 控制台测试

命令行测试指的是测试作为应用程序或包代码库一部分的自定义 Ace 命令。

在本指南中，我们将学习如何为命令编写测试、模拟记录器输出以及捕获 CLI 提示。

## 基本示例

让我们首先创建一个名为 `greet` 的新命令。

```sh
node ace make:command greet
```

```ts
import { BaseCommand } from '@adonisjs/core/ace'
import { CommandOptions } from '@adonisjs/core/types/ace'

export default class Greet extends BaseCommand {
  static commandName = 'greet'
  static description = '按名称问候用户名'

  static options: CommandOptions = {}

  async run() {
    this.logger.info('来自 "Greet" 的 Hello world')
  }
}
```

让我们在 `tests/unit` 目录内创建一个**单元**测试。如果尚未定义，请随意[定义单元测试套件](./introduction.md#suites)。

```sh
node ace make:test commands/greet --suite=unit

# DONE:    create tests/unit/commands/greet.spec.ts
```

让我们打开新创建的文件并编写以下测试。我们将使用 `ace` 服务创建 `Greet` 命令的实例，并断言它成功退出。

```ts
import { test } from '@japa/runner'
import Greet from '#commands/greet'
import ace from '@adonisjs/core/services/ace'

test.group('命令 greet', () => {
  test('应该问候用户并以退出代码 0 结束', async () => {
    /**
     * 创建 Greet 命令类的实例
     */
    const command = await ace.create(Greet, [])

    /**
     * 执行命令
     */
    await command.exec()

    /**
     * 断言命令以状态代码 0 退出
     */
    command.assertSucceeded()
  })
})
```

让我们使用以下 ace 命令运行测试。

```sh
node ace test --files=commands/greet
```

## 测试记录器输出

`Greet` 命令当前将日志消息写入终端。要捕获此消息并为其编写断言，我们必须将 ace 的 UI 库切换到 `raw` 模式。

在 `raw` 模式下，ace 不会将任何日志写入终端。相反，将它们保存在内存中以编写断言。

我们将使用 Japa `each.setup` 钩子来切换进出 `raw` 模式。

```ts
test.group('命令 greet', (group) => {
  // highlight-start
  group.each.setup(() => {
    ace.ui.switchMode('raw')
    return () => ace.ui.switchMode('normal')
  })
  // highlight-end

  // 测试放在这里
})
```

定义钩子后，你可以按如下方式更新测试。

```ts
test('应该问候用户并以退出代码 1 结束', async () => {
  /**
   * 创建 Greet 命令类的实例
   */
  const command = await ace.create(Greet, [])

  /**
   * 执行命令
   */
  await command.exec()

  /**
   * 断言命令以退出代码 0 退出
   */
  command.assertSucceeded()

  // highlight-start
  /**
   * 断言命令打印以下日志消息
   */
  command.assertLog('[ blue(info) ] 来自 "Greet" 的 Hello world')
  // highlight-end
})
```

## 测试表格输出

与测试日志消息类似，你可以通过将 UI 库切换到 `raw` 模式来编写表格输出的断言。

```ts
async run() {
  const table = this.ui.table()
  table.head(['名称', '电子邮件'])

  table.row(['Harminder Virk', 'virk@adonisjs.com'])
  table.row(['Romain Lanz', 'romain@adonisjs.com'])
  table.row(['Julien-R44', 'julien@adonisjs.com'])
  table.row(['Michaël Zasso', 'targos@adonisjs.com'])

  table.render()
}
```

鉴于上述表格，你可以按如下方式为其编写断言。

```ts
const command = await ace.create(Greet, [])
await command.exec()

command.assertTableRows([
  ['Harminder Virk', 'virk@adonisjs.com'],
  ['Romain Lanz', 'romain@adonisjs.com'],
  ['Julien-R44', 'julien@adonisjs.com'],
  ['Michaël Zasso', 'targos@adonisjs.com'],
])
```

## 捕获提示

由于[提示](../ace/prompts.md)会阻塞终端等待手动输入，因此在编写测试时必须以编程方式捕获并响应它们。

提示使用 `prompt.trap` 方法捕获。该方法接受提示标题（区分大小写）并提供可链式调用的 API 来配置其他行为。

陷阱在提示被触发后会自动删除。如果测试完成而没有触发带有陷阱的提示，将引发错误。

在以下示例中，我们在标题为 `"What is your name?"` 的提示上放置陷阱，并使用 `replyWith` 方法回答它。

```ts
const command = await ace.create(Greet, [])

// highlight-start
command.prompt.trap('What is your name?').replyWith('Virk')
// highlight-end

await command.exec()

command.assertSucceeded()
```

### 选择选项

你可以使用 `chooseOption` 和 `chooseOptions` 方法选择选择或多选提示的选项。

```ts
command.prompt.trap('Select package manager').chooseOption(0)
```

```ts
command.prompt.trap('Select database manager').chooseOptions([1, 2])
```

### 接受或拒绝确认提示

你可以使用 `toggle` 和 `confirm` 方法接受或拒绝显示的提示。

```ts
command.prompt.trap('Want to delete all files?').accept()
```

```ts
command.prompt.trap('Want to delete all files?').reject()
```

### 针对验证进行断言

要测试提示的验证行为，你可以使用 `assertPasses` 和 `assertFails` 方法。这些方法接受提示的值，并根据[提示的验证](../ace/prompts.md#prompt-options)方法进行测试。

```ts
command.prompt
  .trap('What is your name?')
  // 断言在提供空值时提示失败
  .assertFails('', 'Please enter your name')

command.prompt.trap('What is your name?').assertPasses('Virk')
```

以下是同时使用断言和回复提示的示例。

```ts
command.prompt
  .trap('What is your name?')
  .assertFails('', 'Please enter your name')
  .assertPasses('Virk')
  .replyWith('Romain')
```

## 可用断言

以下是命令实例上可用的断言方法列表。

### assertSucceeded

断言命令以 `exitCode=0` 退出。

```ts
await command.exec()
command.assertSucceeded()
```

### assertFailed

断言命令以非零 `exitCode` 退出。

```ts
await command.exec()
command.assertFailed()
```

### assertExitCode

断言命令以特定的 `exitCode` 退出。

```ts
await command.exec()
command.assertExitCode(2)
```

### assertNotExitCode

断言命令以任何 `exitCode` 退出，但不是给定的退出代码。

```ts
await command.exec()
command.assertNotExitCode(0)
```

### assertLog

断言命令使用 `this.logger` 属性写入日志消息。你可以选择将输出流断言为 `stdout` 或 `stderr`。

```ts
await command.exec()

command.assertLog('来自 "Greet" 的 Hello world')
command.assertLog('来自 "Greet" 的 Hello world', 'stdout')
```

### assertLogMatches

断言命令写入与给定正则表达式匹配的日志消息。

```ts
await command.exec()

command.assertLogMatches(/Hello world/)
```

### assertTableRows

断言命令将表格打印到 `stdout`。你可以将表格行作为列数组提供。列表示为单元格数组。

```ts
await command.exec()

command.assertTableRows([
  ['Harminder Virk', 'virk@adonisjs.com'],
  ['Romain Lanz', 'romain@adonisjs.com'],
  ['Julien-R44', 'julien@adonisjs.com'],
])
```
