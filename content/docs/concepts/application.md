---
summary: Learn about the Application class and how to access the environment, state, and make URLs and paths to project files.
---

# Application

[Application](https://github.com/adonisjs/application/blob/main/src/application.ts) 类负责组装 AdonisJS 应用程序的所有核心工作。您可以使用此类来了解应用程序运行的环境、获取当前应用状态，或生成项目文件的 URL 和路径。

另请参阅：[应用程序生命周期](./application_lifecycle.md)

## 环境

环境指的是应用程序的运行时环境。应用程序始终在以下已知环境中启动：

- `web` 环境指为 HTTP 服务器启动的进程。

- `console` 环境指除 REPL 命令外的所有 Ace 命令。

- `repl` 环境指通过 `node ace repl` 命令启动的进程。

- 最后, `test` 环境指通过 `node ace test` 命令启动的进程。

您可以使用 `getEnvironment` 方法访问应用程序环境：

```ts
import app from '@adonisjs/core/services/app'

console.log(app.getEnvironment())
```

您也可以在应用程序启动前切换其运行环境。一个典型的例子是 REPL 命令。

`node ace repl` 命令最初以`console` 环境启动应用程序，但该命令内部会在显示 REPL 提示之前将环境切换为 `repl`

```ts
if (!app.isBooted) {
  app.setEnvironment('repl')
}
```

## Node 环境

您可以使用 `nodeEnvironment` 属性访问 Node.js 环境。该值引用的是 `NODE_ENV` 环境变量，但经过进一步规范化以保持一致性：

```ts
import app from '@adonisjs/core/services/app'

console.log(app.nodeEnvironment)
```

| NODE_ENV | 标准化      |
| -------- | ----------- |
| dev      | development |
| develop  | development |
| stage    | staging     |
| prod     | production  |
| testing  | test        |

此外，您还可以使用以下属性作为快捷方式来判断当前环境：

- `inProduction`: 检查应用程序是否在生产环境中运行。
- `inDev`: 检查应用程序是否在开发环境中运行。
- `inTest`: 检查应用程序是否在测试环境中运行。

```ts
import app from '@adonisjs/core/services/app'

// 是否处于生产环境
app.inProduction
app.nodeEnvironment === 'production'

// 是否处于开发环境
app.inDev
app.nodeEnvironment === 'development'

// 是否处于测试环境
app.inTest
app.nodeEnvironment === 'test'
```

## 状态

状态指的是应用程序的当前状态。您能使用的框架功能很大程度上取决于应用程序的当前状态。例如，在应用进入 `booted`(已启动) 状态之前，您无法访问 [容器绑定](./dependency_injection.md#container-bindings) 或 [容器服务](./container_services.md)

应用程序始终处于以下已知状态之一：

- `created`: 这是应用程序的默认状态。

- `initiated`: 在此状态下，我们解析/验证环境变量并处理 `adonisrc.ts` 文件.

- `booted`: 在此状态下，应用程序的服务提供者被注册并启动。

- `ready`: 就绪状态因不同环境而异。例如，在 `web` 环境中，“就绪”意味着应用程序已准备好接受新的 HTTP 请求。

- `terminated`:应用程序已被终止，进程即将退出。在 `web` 环境中，应用程序将不再接受新的 HTTP 请求。

```ts
import app from '@adonisjs/core/services/app'

console.log(app.getState())
```

您也可以使用以下简写属性来判断应用程序是否处于某个特定状态：

```ts
import app from '@adonisjs/core/services/app'

// 应用已启动
app.isBooted
app.getState() !== 'created' && app.getState() !== 'initiated'

// 应用已就绪
app.isReady
app.getState() === 'ready'

// 正在尝试优雅地终止应用
app.isTerminating

// 应用已终止
app.isTerminated
app.getState() === 'terminated'
```

## 监听进程信号

您可以使用 `app.listen` 或 `app.listenOnce` 方法监听 [ POSIX 信号](https://man7.org/linux/man-pages/man7/signal.7.html) 底层实现中，我们将监听器注册到 Node.js 的 `process` 对象上。

```ts
import app from '@adonisjs/core/services/app'

// 监听 SIGTERM 信号
app.listen('SIGTERM', () => {})

// 一次性监听 SIGTERM 信号
app.listenOnce('SIGTERM', () => {})
```

有时，您可能希望有条件地注册监听器。例如，仅当在 pm2 环境中运行时才监听 `SIGINT` 信号。

您可以使用 `listenIf` 或 `listenOnceIf` 方法有条件地注册监听器。只有当第一个参数的值为真值时，监听器才会被注册。

```ts
import app from '@adonisjs/core/services/app'

app.listenIf(app.managedByPm2, 'SIGTERM', () => {})

app.listenOnceIf(app.managedByPm2, 'SIGTERM', () => {})
```

## 通知父进程

如果您的应用程序作为子进程启动，可以使用 `app.notify` 方法向父进程发送消息。底层使用的是`process.send` 方法。

```ts
import app from '@adonisjs/core/services/app'

app.notify('ready')

app.notify({
  isReady: true,
  port: 3333,
  host: 'localhost',
})
```

## 生成项目文件的 URL 和路径

我们强烈建议不要手动构造绝对 URL 或项目文件路径，而是使用以下辅助方法。

### makeURL

makeURL 方法返回指向项目根目录内指定文件或目录的文件 URL。例如，您可以在导入文件时生成 URL。

```ts
import app from '@adonisjs/core/services/app'

const files = ['./tests/welcome.spec.ts', './tests/maths.spec.ts']

await Promise.all(
  files.map((file) => {
    return import(app.makeURL(file).href)
  }),
)
```

### makePath

`makePath` 方法返回指向项目根目录内指定文件或目录的绝对路径。

```ts
import app from '@adonisjs/core/services/app'

app.makePath('app/middleware/auth.ts')
```

### configPath

返回配置目录中某个文件的路径。

```ts
app.configPath('shield.ts')
// /project_root/config/shield.ts

app.configPath()
// /project_root/config
```

### publicPath

返回公共目录中某个文件的路径。

```ts
app.publicPath('style.css')
// /project_root/public/style.css

app.publicPath()
// /project_root/public
```

### providersPath

返回 Providers 目录中某个文件的路径。

```ts
app.providersPath('app_provider')
// /project_root/providers/app_provider.ts

app.providersPath()
// /project_root/providers
```

### factoriesPath

返回数据库工厂目录中某个文件的路径。

```ts
app.factoriesPath('user.ts')
// /project_root/database/factories/user.ts

app.factoriesPath()
// /project_root/database/factories
```

### migrationsPath

返回数据库迁移目录中某个文件的路径。

```ts
app.migrationsPath('user.ts')
// /project_root/database/migrations/user.ts

app.migrationsPath()
// /project_root/database/migrations
```

### seedersPath

返回数据库种子目录中某个文件的路径。

```ts
app.seedersPath('user.ts')
// /project_root/database/seeders/users.ts

app.seedersPath()
// /project_root/database/seeders
```

### languageFilesPath

返回语言文件目录中某个文件的路径。

```ts
app.languageFilesPath('en/messages.json')
// /project_root/resources/lang/en/messages.json

app.languageFilesPath()
// /project_root/resources/lang
```

### viewsPath

返回视图目录中某个文件的路径。

```ts
app.viewsPath('welcome.edge')
// /project_root/resources/views/welcome.edge

app.viewsPath()
// /project_root/resources/views
```

### startPath

返回启动目录中某个文件的路径。

```ts
app.startPath('routes.ts')
// /project_root/start/routes.ts

app.startPath()
// /project_root/start
```

### tmpPath

返回项目根目录下 `tmp` 目录中某个文件的路径。

```ts
app.tmpPath('logs/mail.txt')
// /project_root/tmp/logs/mail.txt

app.tmpPath()
// /project_root/tmp
```

### httpControllersPath

返回 HTTP 控制器目录中某个文件的路径。

```ts
app.httpControllersPath('users_controller.ts')
// /project_root/app/controllers/users_controller.ts

app.httpControllersPath()
// /project_root/app/controllers
```

### modelsPath

返回模型目录中某个文件的路径。

```ts
app.modelsPath('user.ts')
// /project_root/app/models/user.ts

app.modelsPath()
// /project_root/app/models
```

### servicesPath

返回服务目录中某个文件的路径。

```ts
app.servicesPath('user.ts')
// /project_root/app/services/user.ts

app.servicesPath()
// /project_root/app/services
```

### exceptionsPath

返回异常目录中某个文件的路径。

```ts
app.exceptionsPath('handler.ts')
// /project_root/app/exceptions/handler.ts

app.exceptionsPath()
// /project_root/app/exceptions
```

### mailsPath

返回邮件目录中某个文件的路径。

```ts
app.mailsPath('verify_email.ts')
// /project_root/app/mails/verify_email.ts

app.mailsPath()
// /project_root/app/mails
```

### middlewarePath

返回中间件目录中某个文件的路径。

```ts
app.middlewarePath('auth.ts')
// /project_root/app/middleware/auth.ts

app.middlewarePath()
// /project_root/app/middleware
```

### policiesPath

返回策略目录中某个文件的路径。

```ts
app.policiesPath('posts.ts')
// /project_root/app/polices/posts.ts

app.policiesPath()
// /project_root/app/polices
```

### validatorsPath

返回验证器目录中某个文件的路径。

```ts
app.validatorsPath('create_user.ts')
// /project_root/app/validators/create_user.ts

app.validatorsPath()
// /project_root/app/validators/create_user.ts
```

### commandsPath

返回命令目录中某个文件的路径。

```ts
app.commandsPath('greet.ts')
// /project_root/commands/greet.ts

app.commandsPath()
// /project_root/commands
```

### eventsPath

返回事件目录中某个文件的路径。

```ts
app.eventsPath('user_created.ts')
// /project_root/app/events/user_created.ts

app.eventsPath()
// /project_root/app/events
```

### listenersPath

返回监听器目录中某个文件的路径。

```ts
app.listenersPath('send_invoice.ts')
// /project_root/app/listeners/send_invoice.ts

app.listenersPath()
// /project_root/app/listeners
```

## 生成器

生成器用于为不同实体创建类名和文件名。例如，您可以使用 `generators.controllerFileName` 方法为控制器生成文件名。

```ts
import app from '@adonisjs/core/services/app'

app.generators.controllerFileName('user')
// output - users_controller.ts

app.generators.controllerName('user')
// output - UsersController
```

请参考 [ `generators.ts`源码 ](https://github.com/adonisjs/application/blob/main/src/generators.ts) 查看所有可用生成器的列表。
