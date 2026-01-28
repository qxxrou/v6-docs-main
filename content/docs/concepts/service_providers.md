---
summary: Service providers are plain JavaScript classes with lifecycle methods to perform actions during different phases of the application.
---

# Service providers

服务提供者是带有生命周期方法的普通 JavaScript 类，用于在应用程序的不同阶段执行操作。

服务提供者可以在容器中注册[绑定](../concepts/dependency_injection.md#container-bindings)、[扩展现有绑定](../concepts/dependency_injection.md#container-events)，或在 HTTP 服务器启动后运行操作。

服务提供者是 AdonisJS 应用程序的入口点，能够在应用程序被视为准备就绪之前修改其状态。**它主要被外部包用于挂钩到应用程序生命周期中**。

:::note
如果你只想向你的某个类注入依赖项，你可以使用[依赖注入](../concepts/dependency_injection.md)功能。
(../concepts/dependency_injection.md) 功能.
:::

提供者在 `adonisrc.ts` 文件的 `providers` 数组中注册。该值是一个用于延迟导入服务提供者的函数。

```ts
{
  providers: [
    () => import('@adonisjs/core/providers/app_provider'),
    () => import('./providers/app_provider.js'),
  ]
}
```

默认情况下，提供者会在所有运行时环境中加载。不过，你可以将提供者限制在特定环境中运行。

```ts
{
  providers: [
    () => import('@adonisjs/core/providers/app_provider'),
    {
      file: () => import('./providers/app_provider.js'),
      environment: ['web', 'repl'],
    },
  ]
}
```

## 编写服务提供者

服务提供者存储在应用程序的 `providers` 目录中。或者，你可以使用 `node ace make:provider app` 命令创建。

提供者模块必须有一个 `export default` 语句返回提供者类。该类的构造函数接收 [Application](./application.md) 类的实例。

另见：[Make provider 命令](../references/commands.md#makeprovider)

```ts
import { ApplicationService } from '@adonisjs/core/types'

export default class AppProvider {
  constructor(protected app: ApplicationService) {}
}
```

以下是你可以实现的生命周期方法，用于执行不同的操作。

```ts
import { ApplicationService } from '@adonisjs/core/types'

export default class AppProvider {
  constructor(protected app: ApplicationService) {}
}
```

以下是你可以实现的生命周期方法，用于执行不同的操作。

```ts
export default class AppProvider {
  register() {}

  async boot() {}

  async start() {}

  async ready() {}

  async shutdown() {}
}
```

### register

`register` 方法在创建提供者类的实例后被调用。`register` 方法可以在 IoC 容器中注册绑定。

`register` 方法是同步的，因此你不能在该方法内使用 Promise。

```ts
export default class AppProvider {
  register() {
    this.app.container.bind('db', () => {
      return new Database()
    })
  }
}
```

### boot

`boot` 方法在所有绑定都已注册到 IoC 容器后被调用。在这个方法中，你可以从容器中解析绑定以扩展/修改它们。

```ts
export default class AppProvider {
  async boot() {
    const validator = await this.app.container.make('validator')

    // Add custom validation rules
    validator.rule('foo', () => {})
  }
}
```

一个好的做法是在从容器中解析绑定时扩展它们。例如，你可以使用 `resolving` 钩子为验证器添加自定义规则。

```ts
async boot() {
  this.app.container.resolving('validator', (validator) => {
    validator.rule('foo', () => {})
  })
}
```

### start

`start` 方法在 `boot` 之后、`ready` 方法之前被调用。它允许你执行 `ready` 钩子操作可能需要的操作。

### ready

`ready` 方法会根据应用程序的环境在不同阶段被调用。

<table>
    <tr>
        <td width="100"><code> web </code></td>
        <td><code>ready</code> 方法在 HTTP 服务器启动并准备好接受请求后被调用。</td>
    </tr>
    <tr>
        <td width="100"><code>console</code></td>
        <td><code>ready</code> 方法在主命令的 <code>run</code> 方法之前被调用。</td>
    </tr>
    <tr>
        <td width="100"><code>test</code></td>
        <td><code>ready</code> 方法在运行所有测试之前被调用。但是，测试文件在 <code>ready</code> 方法之前被导入。</td>
    </tr>
    <tr>
        <td width="100"><code>repl</code></td>
        <td><code>ready</code> 方法在终端显示 REPL 提示符之前被调用。</td>
    </tr>
</table>

```ts
export default class AppProvider {
  async start() {
    if (this.app.getEnvironment() === 'web') {
    }

    if (this.app.getEnvironment() === 'console') {
    }

    if (this.app.getEnvironment() === 'test') {
    }

    if (this.app.getEnvironment() === 'repl') {
    }
  }
}
```

### shutdown

`shutdown` 方法在 AdonisJS 优雅退出应用程序时被调用。

退出应用程序的事件取决于应用程序运行的环境以及应用程序进程的启动方式。请阅读[应用程序生命周期指南](./application_lifecycle.md)以了解更多信息。

```ts
export default class AppProvider {
  async shutdown() {
    // perform the cleanup
  }
}
```
