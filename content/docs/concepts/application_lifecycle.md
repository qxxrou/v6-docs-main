---
summary: Learn how AdonisJS boots your application and what lifecycle hooks you can use to change the application state before it is considered ready.
---

# 应用程序生命周期

在本指南中，我们将学习 AdonisJS 如何启动您的应用程序，以及可以使用哪些生命周期钩子在应用程序就绪之前改变其状态。

应用程序的生命周期取决于它所运行的环境。例如，用于处理 HTTP 请求的长时运行进程与短时运行的 Ace 命令的管理方式不同。

应用程序的生命周期取决于它所运行的环境。例如，用于处理 HTTP 请求的长时运行进程与短时运行的 Ace 命令的管理方式不同。

## AdonisJS 应用程序是如何启动的

AdonisJS 应用程序有多个入口点，每个入口点会以特定环境启动应用程序。以下入口文件存储在 `bin` 目录中：

- `bin/server.ts` 入口点启动 AdonisJS 应用程序以处理 HTTP 请求。当您运行 `node ace serve` 命令时，幕后我们就是作为子进程运行此文件。
- `bin/console.ts` 入口点启动 AdonisJS 应用程序以处理 CLI 命令。该文件底层使用 [Ace](../ace/introduction.md)
- `bin/test.ts` 入口点启动 AdonisJS 应用程序以使用 Japa 运行测试。

如果您打开这些文件中的任何一个，会发现我们使用了 [Ignitor](https://github.com/adonisjs/core/blob/main/src/ignitor/main.ts#L23) 模块来组装并启动应用程序。

Ignitor 模块封装了启动 AdonisJS 应用程序的逻辑。其内部执行以下操作：

- 创建 [Application](https://github.com/adonisjs/application/blob/main/src/application.ts) 类的实例。
- 初始化/启动应用程序。
- 执行主操作以启动应用程序。例如，在 HTTP 服务器的情况下，“主操作”涉及启动 HTTP 服务器；而在测试的情况下，则是运行测试

[Ignitor codebase](https://github.com/adonisjs/core/tree/main/src/ignitor) 相对简单，建议浏览源码以更好地理解。

## 启动阶段

除了 `console` 环境外，所有环境的启动阶段都相同。在 `console` 环境中，是否启动应用程序由执行的命令决定

只有当应用程序启动后，才能使用容器绑定和服务。

![](./boot_phase_flow_chart.png)

## 启动过程阶段

启动过程阶段在所有环境中各不相同，并且执行流程进一步分为以下子阶段：

- `pre-start` 阶段：指启动应用前执行的操作。

- `post-start` 阶段：指启动应用后执行的操作。对于 HTTP 服务器来说，这些操作将在 HTTP 服务器准备好接受新连接后执行。

![](./start_phase_flow_chart.png)

### 在 Web 环境下

在 Web 环境中，会创建一个长时存在的 HTTP 连接来监听传入请求，应用程序将保持在 `ready` 状态，直到服务器崩溃或进程收到关闭信号为止。

### 在测试环境下

pre-start 和 post-start 操作会在测试环境中执行。之后，我们会导入测试文件并运行测试。

### 在 console 环境下

在 `console` 环境中，是否启动应用程序由执行的命令决定。

命令可以通过启用 `options.startApp` 标志来启动应用程序。这样，pre-start 和 post-start 操作将在命令的 `run` 方法之前运行

```ts
import { BaseCommand } from '@adonisjs/core/ace'

export default class GreetCommand extends BaseCommand {
  static options = {
    startApp: true,
  }

  async run() {
    console.log(this.app.isReady) // true
  }
}
```

## 终止阶段

短时进程和长时进程的应用程序终止方式差异很大。

短时运行的命令或测试进程在主要操作结束后即开始终止。

长时运行的 HTTP 服务器进程则等待退出信号（如`SIGTERM`）来启动终止过程。

![](./termination_phase_flow_chart.png)

### 响应进程信号

在所有环境中，当应用程序接收到 `SIGTERM` 信号时，都会开始优雅关闭过程。如果使用[pm2] (https://pm2.keymetrics.io/docs/usage/signals-clean-restart/), 启动应用程序，则会在接收到 `SIGINT` 事件后触发优雅关闭。

### 在 Web 环境下

在 Web 环境中，应用程序会持续运行，直到底层 HTTP 服务器因错误而崩溃。在这种情况下，我们开始终止应用程序。

### 在测试环境下

当所有测试执行完毕后，开始优雅终止。

### 在控制台环境下

在 `console` 环境中，应用程序的终止取决于执行的命令。

除非启用了 `options.staysAlive` 标志，否则命令执行完成后应用程序立即终止。在此情况下，命令应显式调用终止方法。

```ts
import { BaseCommand } from '@adonisjs/core/ace'

export default class GreetCommand extends BaseCommand {
  static options = {
    startApp: true,
    staysAlive: true,
  }

  async run() {
    await runSomeProcess()

    // Terminate the process
    await this.terminate()
  }
}
```

## 生命周期钩子

生命周期钩子允许您挂接到应用程序的引导过程中，并在应用程序经历不同状态时执行操作。

您可以使用服务提供者类监听钩子，也可以直接在应用程序类上内联定义。

### 内联回调

应在创建应用程序实例后立即注册生命周期钩子。

入口文件 `bin/server.ts`, `bin/console.ts`, 和 `bin/test.ts` 为不同环境创建新的应用程序实例，您可以在这些文件中注册内联回调。

```ts
const app = new Application(new URL('../', import.meta.url))

new Ignitor(APP_ROOT, { importer: IMPORTER }).tap((app) => {
  // highlight-start
  app.booted(() => {
    console.log('invoked after the app is booted')
  })

  app.ready(() => {
    console.log('invoked after the app is ready')
  })

  app.terminating(() => {
    console.log('invoked before the termination starts')
  })
  // highlight-end
})
```

- `initiating`：在应用程序进入`initiating`.状态之前调用钩子操作。 `adonisrc.ts` 文件在执行 initiating 钩子后被解析。

- `booting`: 在启动应用程序之前调用钩子操作。配置文件在执行 `booting` 钩子后被导入。

- `booted`: 在所有服务提供者注册并启动后调用钩子操作。

- `starting`: 在导入预加载文件之前调用钩子操作。

- `ready`: 在应用程序就绪后调用钩子操作。

- `terminating`: 在开始优雅退出流程时调用钩子操作。例如，此钩子可用于关闭数据库连接或结束打开的流。

### 使用服务提供者

服务提供者在提供者类中将生命周期钩子定义为方法。我们推荐使用服务提供者而非内联回调，因为它能更清晰地组织代码。

以下是可用的生命周期方法列表：

```ts
import { ApplicationService } from '@adonisjs/core/types'

export default class AppProvider {
  constructor(protected app: ApplicationService) {}

  register() {}

  async boot() {}

  async start() {}

  async ready() {}

  async shutdown() {}
}
```

- `register`: 方法用于在容器中注册绑定。该方法设计为同步执行。

- `boot`: 方法用于启动或初始化您在容器中注册的绑定。

- `start`: 方法在 `ready`方法之前运行。允许您执行 `ready` 钩子可能需要的操作。

- `ready`: 方法在应用程序被认为就绪后运行。

- `shutdown`: shutdown 方法在应用程序开始优雅关闭时被调用。可用于关闭数据库连接或结束已打开的流。
