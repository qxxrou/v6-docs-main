---
summary: 学习如何在 AdonisJS 中创建自定义 Ace 命令
---

# 创建命令

除了使用 Ace 命令外，您还可以创建自定义命令作为应用程序代码库的一部分。命令存储在 `commands` 目录中（根级别）。您可以通过运行以下命令来创建命令。

另请参阅：[Make command](../references/commands.md#makecommand)

```sh
node ace make:command greet
```

上述命令将在 `commands` 目录中创建 `greet.ts` 文件。Ace 命令由类表示，必须实现 `run` 方法来执行命令指令。

## 命令元数据

命令元数据包括**命令名称**、**描述**、**帮助文本**和配置命令行为的**选项**。

```ts
import { BaseCommand } from '@adonisjs/core/ace'
import { CommandOptions } from '@adonisjs/core/types/ace'

export default class GreetCommand extends BaseCommand {
  static commandName = 'greet'
  static description = '按名称问候用户'

  static options: CommandOptions = {
    startApp: false,
    allowUnknownFlags: false,
    staysAlive: false,
  }
}
```

<dl>
<dt>
  commandName
</dt>

<dd>

`commandName` 属性用于定义命令名称。命令名称不应包含空格。此外，建议避免在命令名称中使用不熟悉特殊字符，如 `*`、`&` 或斜杠。

命令名称可以位于命名空间下。例如，要在 `make` 命名空间下定义命令，您可以为其添加 `make:` 前缀。

</dd>

<dt>
  description
</dt>

<dd>

命令描述显示在命令列表和命令帮助屏幕中。您必须保持描述简短，并使用**帮助文本**作为更长的描述。

</dd>

<dt>
  help
</dt>

<dd>

帮助文本用于编写更长的描述或显示用法示例。

```ts
export default class GreetCommand extends BaseCommand {
  static help = [
    'greet 命令用于按名称问候用户',
    '',
    '如果用户有更新的地址，您也可以向他们发送鲜花',
    '{{ binaryName }} greet --send-flowers',
  ]
}
```

> `{{ binaryName }}` 变量替换是对用于执行 ace 命令的二进制文件的引用。

</dd>

<dt>
  aliases
</dt>

<dd>

您可以使用 `aliases` 属性为命令名称定义一个或多个别名。

```ts
export default class GreetCommand extends BaseCommand {
  static commandName = 'greet'
  static aliases = ['welcome', 'sayhi']
}
```

</dd>

<dt>
  options.startApp
</dt>

<dd>

默认情况下，AdonisJS 在运行 Ace 命令时不会启动应用程序。这确保命令运行快速，并且不会经历应用程序启动阶段来执行简单任务。

但是，如果您的命令依赖于应用程序状态，您可以告诉 Ace 在执行命令之前启动应用程序。

```ts
import { BaseCommand } from '@adonisjs/core/ace'
import { CommandOptions } from '@adonisjs/core/types/ace'

export default class GreetCommand extends BaseCommand {
  // highlight-start
  static options: CommandOptions = {
    startApp: true,
  }
  // highlight-end
}
```

</dd>

<dt>
  options.allowUnknownFlags
</dt>

<dd>

默认情况下，如果您向命令传递未知标志，Ace 会打印错误。但是，您可以使用 `options.allowUnknownFlags` 属性在命令级别禁用严格的标志解析。

```ts
import { BaseCommand } from '@adonisjs/core/ace'
import { CommandOptions } from '@adonisjs/core/types/ace'

export default class GreetCommand extends BaseCommand {
  // highlight-start
  static options: CommandOptions = {
    allowUnknownFlags: true,
  }
  // highlight-end
}
```

</dd>

<dt>
  options.staysAlive
</dt>

<dd>

AdonisJS 在执行命令的 `run` 命令后隐式终止应用程序。但是，如果您想在命令中启动长时间运行的进程，您必须告诉 Ace 不要终止进程。

另请参阅：[终止应用程序](#terminating-application) 和 [在应用程序终止前清理](#cleaning-up-before-the-app-terminates) 部分。

```ts
import { BaseCommand } from '@adonisjs/core/ace'
import { CommandOptions } from '@adonisjs/core/types/ace'

export default class GreetCommand extends BaseCommand {
  // highlight-start
  static options: CommandOptions = {
    staysAlive: true,
  }
  // highlight-end
}
```

</dd>
</dl>

## 命令生命周期方法

您可以在命令类上定义以下生命周期方法，Ace 将按预定义顺序执行它们。

```ts
import { BaseCommand, args, flags } from '@adonisjs/core/ace'

export default class GreetCommand extends BaseCommand {
  async prepare() {
    console.log('准备中')
  }

  async interact() {
    console.log('交互中')
  }

  async run() {
    console.log('运行中')
  }

  async completed() {
    console.log('已完成')
  }
}
```

<table>
  <thead>
    <tr>
      <th width="120px">方法</th>
      <th>描述</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>
        <code>prepare</code>
      </td>
      <td>这是 Ace 在命令上执行的第一个方法。此方法可以设置运行命令所需的状态或数据。</td>
    </tr>
    <tr>
      <td>
        <code>interact</code>
      </td>
      <td>interact 方法在 <code>prepare</code> 方法之后执行。它可以用于向用户显示提示。</td>
    </tr>
    <tr>
      <td>
        <code>run</code>
      </td>
      <td>命令主要逻辑位于 <code>run</code> 方法中。此方法在 <code>interact</code> 方法之后调用。</td>
    </tr>
    <tr>
      <td>
        <code>completed</code>
      </td>
      <td>completed 方法在运行所有其他生命周期方法之后调用。此方法可用于执行清理或处理/显示其他方法抛出的异常。</td>
    </tr>
  </tbody>
</table>

## 依赖注入

Ace 命令是使用 [IoC 容器](../concepts/dependency_injection.md) 构建和执行的。因此，您可以在命令生命周期方法上进行类型提示依赖项，并使用 `@inject` 装饰器来解析它们。

为了演示，让我们在所有生命周期方法中注入 `UserService` 类。

```ts
import { inject } from '@adonisjs/core'
import { BaseCommand } from '@adonisjs/core/ace'
import UserService from '#services/user_service'

export default class GreetCommand extends BaseCommand {
  @inject()
  async prepare(userService: UserService) {}

  @inject()
  async interact(userService: UserService) {}

  @inject()
  async run(userService: UserService) {}

  @inject()
  async completed(userService: UserService) {}
}
```

## 处理错误和退出代码

命令引发的异常将使用 CLI 日志记录器显示，并且命令 `exitCode` 设置为 `1`（非零错误代码意味着命令失败）。

但是，您也可以通过将代码包装在 `try/catch` 块中或使用 `completed` 生命周期方法来捕获错误。在任何情况下，请记住更新命令 `exitCode` 和 `error` 属性。

### 使用 try/catch 处理错误

```ts
import { BaseCommand } from '@adonisjs/core/ace'

export default class GreetCommand extends BaseCommand {
  async run() {
    try {
      await runSomeOperation()
    } catch (error) {
      this.logger.error(error.message)
      this.error = error
      this.exitCode = 1
    }
  }
}
```

### 在 completed 方法内处理错误

```ts
import { BaseCommand } from '@adonisjs/core/ace'

export default class GreetCommand extends BaseCommand {
  async run() {
    await runSomeOperation()
  }

  async completed() {
    if (this.error) {
      this.logger.error(this.error.message)

      /**
       * 通知 Ace 错误已处理
       */
      return true
    }
  }
}
```

## 终止应用程序

默认情况下，Ace 将在执行命令后终止应用程序。但是，如果您启用了 `staysAlive` 选项，则必须使用 `this.terminate` 方法显式终止应用程序。

假设我们建立 redis 连接来监控服务器内存。我们监听 redis 连接上的 `error` 事件，并在连接失败时终止应用程序。

```ts
import { BaseCommand } from '@adonisjs/core/ace'
import { CommandOptions } from '@adonisjs/core/types/ace'

export default class GreetCommand extends BaseCommand {
  static options: CommandOptions = {
    staysAlive: true,
  }

  async run() {
    const redis = createRedisConnection()

    // highlight-start
    redis.on('error', (error) => {
      this.logger.error(error)
      this.terminate()
    })
    // highlight-end
  }
}
```

## 在应用程序终止前清理

多个事件可以触发应用程序终止，包括 [`SIGTERM` 信号](https://www.howtogeek.com/devops/linux-signals-hacks-definition-and-more/)。因此，您必须在命令中监听 `terminating` 钩子来执行清理。

您可以在 `prepare` 生命周期方法中监听终止钩子。

```ts
import { BaseCommand } from '@adonisjs/core/ace'
import { CommandOptions } from '@adonisjs/core/types/ace'

export default class GreetCommand extends BaseCommand {
  static options: CommandOptions = {
    staysAlive: true,
  }

  prepare() {
    this.app.terminating(() => {
      // 执行清理
    })
  }

  async run() {}
}
```
