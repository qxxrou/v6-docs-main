---
summary: 参观 AdonisJS 在安装过程中创建的重要文件和文件夹。
---

# 文件夹结构

在本指南中，我们将参观 AdonisJS 在安装过程中创建的重要文件和文件夹。

我们提供了一个经过深思熟虑的默认文件夹结构，帮助你保持项目整洁且易于重构。然而，你有充分的自由可以偏离，并拥有适合你的团队和项目的文件夹结构。

## `adonisrc.ts` 文件

`adonisrc.ts` 文件用于配置工作区以及应用程序的一些运行时设置。

在此文件中，你可以注册提供商、定义命令别名，或指定要复制到生产构建中的文件。

另请参阅：[AdonisRC 文件参考指南](../concepts/adonisrc_file.md)

## `tsconfig.json` 文件

`tsconfig.json` 文件存储应用程序的 TypeScript 配置。请随意根据你的项目或团队的要求更改此文件。

以下配置选项是 AdonisJS 内部正常工作所必需的。

```json
{
  "compilerOptions": {
    "module": "NodeNext",
    "isolatedModules": true,
    "declaration": false,
    "outDir": "./build",
    "esModuleInterop": true,
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,
    "skipLibCheck": true
  }
}
``` 

## 子路径导入

AdonisJS 使用 Node.js 的[子路径导入](https://nodejs.org/dist/latest-v19.x/docs/api/packages.html#subpath-imports)功能来定义导入别名。

以下导入别名在 `package.json` 文件中预先配置。随意添加新别名或编辑现有别名。

```json
// title: package.json
{
  "imports": {
    "#controllers/*": "./app/controllers/*.js",
    "#exceptions/*": "./app/exceptions/*.js",
    "#models/*": "./app/models/*.js",
    "#mails/*": "./app/mails/*.js",
    "#services/*": "./app/services/*.js",
    "#listeners/*": "./app/listeners/*.js",
    "#events/*": "./app/events/*.js",
    "#middleware/*": "./app/middleware/*.js",
    "#validators/*": "./app/validators/*.js",
    "#providers/*": "./app/providers/*.js",
    "#policies/*": "./app/policies/*.js",
    "#abilities/*": "./app/abilities/*.js",
    "#database/*": "./database/*.js",
    "#tests/*": "./tests/*.js",
    "#start/*": "./start/*.js",
    "#config/*": "./config/*.js"
  }
}
```

## `bin` 目录

`bin` 目录具有在特定环境中加载应用程序的入口点文件。例如：

- `bin/server.ts` 文件在网络环境中启动应用程序以侦听 HTTP 请求。 
- `bin/console.ts` 文件启动 Ace 命令行并执行命令。
- `bin/test.ts` 文件启动应用程序以运行测试。

## `ace.js` 文件

`ace` 文件启动特定于你应用程序的命令行框架。因此，每次你运行 ace 命令时，都会经过此文件。

如果你注意到，ace 文件以 `.js` 扩展名结尾。这是因为我们希望使用 `node` 二进制文件运行此文件，而无需编译它。

## `app` 目录

`app` 目录组织应用程序领域逻辑的代码。例如，控制器、模型、服务等全部位于 `app` 目录内。

随意创建额外的目录以更好地组织你的应用程序代码。
```
├── app
│  └── controllers
│  └── exceptions
│  └── middleware
│  └── models
│  └── validators

```


## `resources` 目录

`resources` 目录包含 Edge 模板以及前端代码的源文件。换句话说，你的应用程序表示层的代码位于 `resources` 目录内。
```
├── resources
│  └── views
│  └── js
│  └── css
│  └── fonts
│  └── images

```

## `start` 目录

`start` 目录包含你希望在应用程序启动生命周期内导入的文件。例如，注册路由和定义事件侦听器的文件应位于 `start` 目录内。
```
├── start
│  ├── env.ts
│  ├── kernel.ts
│  ├── routes.ts
│  ├── validator.ts
│  ├── events.ts

```

AdonisJS 不会自动从 `start` 目录导入文件。它仅用作将类似文件分组的约定。

我们建议阅读有关[预加载文件](../concepts/adonisrc_file.md#preloads)和[应用程序启动生命周期](../concepts/application_lifecycle.md)的内容，以更好地了解哪些文件应保留在 `start` 目录下。

## `public` 目录

`public` 目录托管 CSS 文件、图像、字体或前端 JavaScript 等静态资源。

不要将 `public` 目录与 `resources` 目录混淆。resources 目录包含前端应用程序的源代码，而 public 目录具有编译后的输出。

使用 Vite 时，你应将前端资源存储在 `resources/<SUB_DIR>` 目录内，并让 Vite 编译器在 `public` 目录中创建输出。

另一方面，如果你不使用 Vite，你可以直接在 `public` 目录内创建文件并使用文件名访问它们。例如，你可以从 `http://localhost:3333/style.css` URL 访问 `./public/style.css` 文件。

## `database` 目录

`database` 目录包含数据库迁移和播种器的文件。
```
├── database
│  └── migrations
│  └── seeders

```


## `commands` 目录

[ace 命令](../ace/introduction.md)存储在 `commands` 目录内。你可以通过运行 `node ace make:command` 在此文件夹内创建命令。


## `config` 目录

`config` 目录包含应用程序的运行时配置文件。

框架的核心和其他已安装的包从此目录读取配置文件。你还可以在此目录内存储特定于你应用程序的配置。

了解有关[配置管理](./configuration.md)的更多信息。
```
├── config
│  ├── app.ts
│  ├── bodyparser.ts
│  ├── cors.ts
│  ├── database.ts
│  ├── drive.ts
│  ├── hash.ts
│  ├── logger.ts
│  ├── session.ts
│  ├── static.ts

```


## `types` 目录

`types` 目录是 TypeScript 接口或类型的存放地，这些接口或类型在你的应用程序中使用。 

该目录默认情况下为空，然而，你可以在 `types` 目录内创建文件和文件夹以定义自定义类型和接口。
```
├── types
│  ├── events.ts
│  ├── container.ts

```

## `providers` 目录

`providers` 目录用于存储你的应用程序使用的[服务提供商](../concepts/service_providers.md)。你可以使用 `node ace make:provider` 命令创建新的提供商。

了解有关[服务提供商](../concepts/service_providers.md)的更多信息
```
├── providers
│  └── app_provider.ts
│  └── http_server_provider.ts

```

## `tmp` 目录

你的应用程序生成的临时文件存储在 `tmp` 目录内。例如，这些可能是用户上传的文件（在开发过程中生成）或写入磁盘的日志。

`tmp` 目录必须被 `.gitignore` 规则忽略，你也不应该将其复制到生产服务器。

## `tests` 目录

`tests` 目录组织你的应用程序测试。此外，还为 `unit` 和 `functional` 测试创建子目录。

另请参阅：[测试](../testing/introduction.md)
```
├── tests
│  ├── bootstrap.ts
│  └── functional
│  └── regression
│  └── unit

```
