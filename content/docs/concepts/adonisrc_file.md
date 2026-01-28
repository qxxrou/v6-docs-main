# AdonisRC 文件

该`adonisrc.ts`文件用于配置应用程序的工作区设置。您可以在此文件中[注册提供程序](https://docs.adonisjs.com/guides/concepts/adonisrc-file#providers)、定义[命令别名](https://docs.adonisjs.com/guides/concepts/adonisrc-file#commandsaliases)、指定[要复制](https://docs.adonisjs.com/guides/concepts/adonisrc-file#metafiles)到生产构建的文件等等。

`该adonisrc.ts文件会被 AdonisJS 应用程序以外的其他工具导入。因此，您不得在此文件中编写任何特定于应用程序的代码或特定于环境的条件语句。`

该文件包含运行应用程序所需的最低配置。但是，您可以通过运行`node ace inspect:rcfile`命令来查看完整的文件内容。

```shell
node ace inspect:rcfile
```

您可以使用该服务访问已解析的 RCFile 内容`app`。

```ts
import app from '@adonisjs/core/services/app'

console.log(app.rcFile)
```

## TypeScript

该`typescript`属性告知框架和 Ace 命令您的应用程序使用 TypeScript。目前，此值始终设置为 `true`。但是，我们稍后将允许使用 JavaScript 编写应用程序。

## 目录

一组目录及其路径，供脚手架命令使用。如果您决定重命名特定目录，请更新对象内部的相应路径，`directories`以便通知脚手架命令。

```ts
{
  directories: {
    config: 'config',
    commands: 'commands',
    contracts: 'contracts',
    public: 'public',
    providers: 'providers',
    languageFiles: 'resources/lang',
    migrations: 'database/migrations',
    seeders: 'database/seeders',
    factories: 'database/factories',
    views: 'resources/views',
    start: 'start',
    tmp: 'tmp',
    tests: 'tests',
    httpControllers: 'app/controllers',
    models: 'app/models',
    services: 'app/services',
    exceptions: 'app/exceptions',
    mails: 'app/mails',
    middleware: 'app/middleware',
    policies: 'app/policies',
    validators: 'app/validators',
    events: 'app/events',
    listeners: 'app/listeners',
    stubs: 'stubs',
  }
}

```

## 预加载

应用程序启动时要导入的文件数组。这些文件会在服务提供程序启动后立即导入。

您可以定义导入文件的环境。有效选项包括：

- `web`环境指的是HTTP服务器启动的进程。
- `console`环境指的是除该命令之外的 Ace 命令`repl`。
- `repl`环境指的是使用该命令启动的进程`node ace repl`。
- 最后，`test`环境指的是为运行测试而启动的进程。

您可以使用该命令创建并注册预加载文件`node ace make:preload`。

```ts
{
  preloads: [() => import('./start/view.js')]
}
```

```ts
{
  preloads: [
    {
      file: () => import('./start/view.js'),
      environment: ['web', 'console', 'test'],
    },
  ]
}
```

## 元文件

`metaFiles`该数组是创建生产版本时要复制文件到`build`文件夹中的的集合。

这些是非 TypeScript/JavaScript 文件，必须存在于生产版本中才能使应用程序正常运行。例如，Edge 模板、i18n 语言文件等。

- pattern：用于查找匹配文件的[glob 模式](https://github.com/sindresorhus/globby#globbing-patterns)。
- reloadServer当匹配文件发生更改时，重新加载开发服务器。

```ts
{
  metaFiles: [
    {
      pattern: 'public/**',
      reloadServer: false,
    },
    {
      pattern: 'resources/views/**/*.edge',
      reloadServer: false,
    },
  ]
}
```

## 命令

一个用于从已安装软件包中延迟导入 ace 命令的函数数组。您的应用程序命令将自动导入，因此您无需显式注册它们。

另请参阅：[创建 ace 命令](https://docs.adonisjs.com/guides/ace/creating-commands)

```ts
{
  commands: [
    () => import('@adonisjs/core/commands'),
    () => import('@adonisjs/lucid/commands'),
  ]
}
```

## 命令别名

命令别名采用键值对的形式。这通常用于帮助您为那些难以输入或记忆的命令创建易于记忆的别名。

另请参阅：[创建命令别名](https://docs.adonisjs.com/guides/ace/introduction#creating-command-aliases)

```ts
{
  commandsAliases: {
    migrate: 'migration:run'
  }
}
```

您还可以为同一命令定义多个别名。

```ts
{
  commandsAliases: {
    migrate: 'migration:run',
    up: 'migration:run'
  }
}

```

## 测试

该`tests`对象注册测试套件和测试运行器的一些全局设置。

另请参阅：[测试简介](https://docs.adonisjs.com/guides/testing/introduction)

```ts
{
  tests: {
    timeout: 2000,
    forceExit: false,
    suites: [
      {
        name: 'functional',
        files: [
          'tests/functional/**/*.spec.ts'
        ],
        timeout: 30000
      }
    ]
  }
}

```

- `timeout`定义所有测试的默认超时时间。
- `forceExit`测试完成后，立即强制退出应用程序进程。通常来说，优雅地退出是一种良好的编程实践。
- `suite.name`：测试套件的唯一名称。
- `suite.files`：用于导入测试文件的 glob 模式数组。
- `suite.timeout`：测试套件中所有测试的默认超时时间。

## providers

在应用程序启动阶段要加载的服务提供程序数组。

默认情况下，提供程序会加载到所有环境中。但是，您也可以定义一个显式的环境数组来导入提供程序。

- `web`环境指的是HTTP服务器启动的进程。
- `console`环境指的是除该命令之外的 Ace 命令`repl`。
- `repl`环境指的是使用该命令启动的进程`node ace repl`。
- 最后，`test`环境指的是为运行测试而启动的进程。

`提供程序按照其在数组中注册的顺序加载providers。`

另请参阅：[服务提供商](https://docs.adonisjs.com/guides/concepts/service-providers)

```ts
{
  providers: [
    () => import('@adonisjs/core/providers/app_provider'),
    () => import('@adonisjs/core/providers/http_provider'),
    () => import('@adonisjs/core/providers/hash_provider'),
    () => import('./providers/app_provider.js'),
  ]
}
```

```ts
{
  providers: [
    {
      file: () => import('./providers/app_provider.js'),
      environment: ['web', 'console', 'test'],
    },
    {
      file: () => import('@adonisjs/core/providers/http_provider'),
      environment: ['web'],
    },
    () => import('@adonisjs/core/providers/hash_provider'),
    () => import('@adonisjs/core/providers/app_provider'),
  ]
}
```

## assetsBundler

`serve`和`build`命令尝试检测应用程序用于编译前端资源的资源。

对[vite](https://vitejs.dev/)和[Webpack encore 的](https://github.com/symfony/webpack-encore)检测分别通过搜索`vite.config.js` 和 `webpack.config.js`文件进行。

但是，如果您使用不同的资源打包器，则可以在文件中按如下方式进行配置adonisrc.ts。

```ts
{
  assetsBundler: {
    name: 'vite',
    devServer: {
      command: 'vite',
      args: []
    },
    build: {
      command: 'vite',
      args: ["build"]
    },
  }
}

```

- `name`- 您使用的资源打包工具的名称。此信息为必填项，用于显示。
- `devServer.\*`- 启动开发服务器的命令及其参数。
- `build.\*`- 用于创建生产版本的命令及其参数。
