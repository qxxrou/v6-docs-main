---
summary: Ace 终端界面利用 @poppinss/cliui 包，提供显示日志、表格和动画的工具。专为测试设计，包含一个"原始"模式以简化日志收集和断言。
---

# 终端界面

Ace 终端界面由 [@poppinss/cliui](https://github.com/poppinss/cliui) 包提供支持。该包导出帮助程序来显示日志、呈现表格、呈现动画任务界面等等。

终端界面原语是考虑到测试而构建的。编写测试时，您可以打开 `raw` 模式来禁用颜色和格式化，并在内存中收集所有日志以针对它们编写断言。

另请参阅：[测试 Ace 命令](../testing/console_tests.md)

## 显示日志消息

您可以使用 CLI 日志记录器显示日志消息。以下是可用的日志方法列表。

```ts
import { BaseCommand } from '@adonisjs/core/ace'

export default class GreetCommand extends BaseCommand {
  async run() {
    this.logger.debug('刚刚发生了一些事情')
    this.logger.info('这是一条信息消息')
    this.logger.success('账户已创建')
    this.logger.warning('磁盘空间不足')

    // 写入 stderr
    this.logger.error(new Error('无法写入。磁盘已满'))
    this.logger.fatal(new Error('无法写入。磁盘已满'))
  }
}
```

### 添加前缀和后缀

使用选项对象，您可以为日志消息定义 `prefix` 和 `suffix`。前缀和后缀以较低的不透明度显示。

```ts
this.logger.info('安装包', {
  suffix: 'npm i --production',
})

this.logger.info('安装包', {
  prefix: process.pid,
})
```

### 加载动画

带有加载动画的日志消息会将动画的三个点 (...) 附加到消息。您可以使用 `animation.update` 方法更新日志消息，并使用 `animation.stop` 方法停止动画。

```ts
const animation = this.logger.await('安装包', {
  suffix: 'npm i',
})

animation.start()

// 更新消息
animation.update('解包', {
  suffix: undefined,
})

// 停止动画
animation.stop()
```

### 日志记录器操作

日志记录器操作可以显示具有一致样式和颜色的操作状态。

您可以使用 `logger.action` 方法创建操作。该方法接受操作标题作为第一个参数。

```ts
const createFile = this.logger.action('创建 config/auth.ts')

try {
  await performTasks()
  createFile.displayDuration().succeeded()
} catch (error) {
  createFile.failed(error)
}
```

您可以将操作标记为 `succeeded`、`failed` 或 `skipped`。

```ts
action.succeeded()
action.skipped('跳过原因')
action.failed(new Error())
```

## 使用 ANSI 颜色格式化文本

Ace 使用 [kleur](https://www.npmjs.com/package/kleur) 来使用 ANSI 颜色格式化文本。使用 `this.colors` 属性，您可以访问 kleur 的链式 API。

```ts
this.colors.red('[错误]')
this.colors.bgGreen().white(' 已创建 ')
```

## 呈现表格

可以使用 `this.ui.table` 方法创建表格。该方法返回一个 `Table` 类的实例，您可以使用它来定义表格头和行。

```ts
import { BaseCommand } from '@adonisjs/core/ace'

export default class GreetCommand extends BaseCommand {
  async run() {
    const table = this.ui.table()

    table
      .head(['迁移', '持续时间', '状态'])
      .row(['1590591892626_tenants.ts', '2ms', '已完成'])
      .row(['1590595949171_entities.ts', '2ms', '已完成'])
      .render()
  }
}
```

您可以在呈现表格时对任何值应用颜色变换。例如：

```ts
table.row(['1590595949171_entities.ts', '2', this.colors.green('已完成')])
```

### 右对齐列

您可以通过将它们定义为对象并使用 hAlign 属性来右对齐列。确保也右对齐标题列。

```ts
table
  .head([
    '迁移',
    '批次'
    {
      content: '状态',
      hAlign: 'right'
    },
  ])

table.row([
  '1590595949171_entities.ts',
  '2',
  {
    content: this.colors.green('已完成'),
    hAlign: 'right'
  }
])
```

### 全宽呈现

默认情况下，表格以宽度 `auto` 呈现，只占用每列内容所需的空间。

但是，您可以使用 `fullWidth` 方法以全宽（占用所有终端空间）呈现表格。在全宽模式下：

- 我们将根据内容大小呈现所有列。
- 除了第一列占用所有可用空间。

```ts
table.fullWidth().render()
```

您可以通过调用 `table.fluidColumnIndex` 方法来更改流动列（占用所有空间的列）的列索引。

```ts
table.fullWidth().fluidColumnIndex(1)
```

## 打印贴纸

贴纸允许您在带有边框的框中呈现内容。当您想将用户的注意力吸引到重要信息时，它们会很有帮助。

您可以使用 `this.ui.sticker` 方法创建贴纸实例。

```ts
import { BaseCommand } from '@adonisjs/core/ace'

export default class GreetCommand extends BaseCommand {
  async run() {
    const sticker = this.ui.sticker()

    sticker
      .add('已启动 HTTP 服务器')
      .add('')
      .add(`本地地址:   ${this.colors.cyan('http://localhost:3333')}`)
      .add(`网络地址: ${this.colors.cyan('http://192.168.1.2:3333')}`)
      .render()
  }
}
```

如果您想在框内显示一组说明，可以使用 `this.ui.instructions` 方法。说明输出中的每一行都将以箭头符号 `>` 为前缀。

## 呈现任务

任务小部件为分享多个耗时任务的进度提供了出色的 UI。该小部件具有以下两种呈现模式：

- 在 `minimal` 模式下，当前运行任务的 UI 会展开以一次显示一个日志行。
- 在 `verbose` 模式下，每个日志语句都在其行中呈现。详细呈现器必须用于调试任务。

您可以使用 `this.ui.tasks` 方法创建任务小部件实例。

```ts
import { BaseCommand } from '@adonisjs/core/ace'

export default class GreetCommand extends BaseCommand {
  async run() {
    const tasks = this.ui.tasks()

    // ... 其余代码如下
  }
}
```

使用 `tasks.add` 方法添加单个任务。该方法接受任务标题作为第一个参数和实现回调作为第二个参数。

您必须从回调中返回任务的状态。所有返回值都被视为成功消息，直到您将它们包装在 `task.error` 方法中或在回调内抛出异常。

```ts
import { BaseCommand } from '@adonisjs/core/ace'

export default class GreetCommand extends BaseCommand {
  async run() {
    const tasks = this.ui.tasks()

    // highlight-start
    await tasks
      .add('克隆仓库', async (task) => {
        return '已完成'
      })
      .add('更新包文件', async (task) => {
        return task.error('无法更新包文件')
      })
      .add('安装依赖项', async (task) => {
        return '已安装'
      })
      .run()
    // highlight-end
  }
}
```

### 报告任务进度

建议调用 `task.update` 方法，而不是将任务进度消息写入控制台。

`update` 方法使用 `minimal` 呈现器显示最新日志消息，并使用 `verbose` 呈现器记录所有消息。

```ts
const sleep = () => new Promise<void>((resolve) => setTimeout(resolve, 50))
const tasks = this.ui.tasks()
await tasks
  .add('克隆仓库', async (task) => {
    for (let i = 0; i <= 100; i = i + 2) {
      await sleep()
      task.update(`已下载 ${i}%`)
    }

    return '已完成'
  })
  .run()
```

### 切换到详细呈现器

创建任务小部件时，您可以切换到详细呈现器。您可以考虑允许命令的用户传递标志来打开 `verbose` 模式。

```ts
import { BaseCommand, flags } from '@adonisjs/core/ace'

export default class GreetCommand extends BaseCommand {
  @flags.boolean()
  declare verbose: boolean

  async run() {
    const tasks = this.ui.tasks({
      verbose: this.verbose,
    })
  }
}
```
