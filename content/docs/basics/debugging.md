---
summary: 探索调试 AdonisJS 应用程序的多种方法，从使用 VSCode 调试器到使用 Dump and Die 以及查看框架的调试日志。
---

# 调试

在本指南中，我们将探索调试 AdonisJS 应用程序的多种方法，从使用 VSCode 调试器到使用 Dump and Die 以及查看框架的调试日志。

## 使用 VSCode 调试器进行调试

使用 VSCode 调试器调试 AdonisJS 应用程序非常简单。您只需要创建一个 `.vscode/launch.json` 文件并使用 Node.js 调试器即可。

在下面的示例中，我们定义了一个配置，用于在调试模式下启动 AdonisJS 开发服务器，然后将 VSCode 调试器附加到它。

另请参阅：[VSCode 调试文档](https://code.visualstudio.com/docs/editor/debugging)

```json
// title: .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Dev server",
      "program": "${workspaceFolder}/ace.js",
      "args": ["serve", "--hmr"],
      "skipFiles": ["<node_internals>/**"]
    }
  ]
}
```

开始调试：

- 使用 `(CMD + Shift + P)` 打开命令面板。
- 搜索 **Debug: Select and Start Debugging**。您将找到来自 `.vscode/launch.json` 文件的启动选项列表。
- 选择 **Dev server** 选项以使用 VSCode 调试器运行 HTTP 服务器。

![](./debug-dev-server.png)

### 调试测试

您可以定义另一个启动选项以在调试模式下运行测试。

```json
// title: .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Dev server",
      "program": "${workspaceFolder}/ace.js",
      "args": ["serve", "--hmr"],
      "skipFiles": ["<node_internals>/**"]
    },
    // insert-start
    {
      "type": "node",
      "request": "launch",
      "name": "Tests",
      "program": "${workspaceFolder}/ace.js",
      "args": ["test", "--watch"],
      "skipFiles": ["<node_internals>/**"]
    }
    // insert-end
  ]
}
```

### 调试所有其他 Ace 命令

为每个 ace 命令定义单独的启动选项不是一个实用的选择。因此，您可以在 `.vscode/launch.json` 文件中定义一个 `attach` 配置。

在 `attach` 模式下，[VSCode 将附加其调试器](https://code.visualstudio.com/blogs/2018/07/12/introducing-logpoints-and-auto-attach#_autoattaching-to-node-processes)到已经在运行的 Node.js 进程，前提是该进程是从 VSCode 集成终端中使用 `--inspect` 标志启动的。

让我们首先修改 `.vscode/launch.json` 文件，并向其中添加以下配置。

```json
// title: .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    // insert-start
    {
      "type": "node",
      "request": "attach",
      "name": "Attach Program",
      "port": 9229,
      "autoAttachChildProcesses": true,
      "skipFiles": ["<node_internals>/**"]
    },
    // insert-end
    {
      "type": "node",
      "request": "launch",
      "name": "Dev server",
      "program": "${workspaceFolder}/ace.js",
      "args": ["serve", "--hmr"],
      "skipFiles": ["<node_internals>/**"]
    },
    {
      "type": "node",
      "request": "launch",
      "name": "Tests",
      "program": "${workspaceFolder}/ace.js",
      "args": ["test", "--watch"],
      "skipFiles": ["<node_internals>/**"]
    }
  ]
}
```

在 attach 模式下开始调试：

- 使用 `(CMD + Shift + P)` 打开命令面板。
- 搜索 **Debug: Select and Start Debugging**。您将找到来自 `.vscode/launch.json` 文件的启动选项列表。
- 选择 **Attach Program** 选项。
- 使用 `--inspect` 标志运行 Ace 命令。例如：
  ```sh
  node --inspect ace migration:run
  ```

::video{url="https://res.cloudinary.com/adonis-js/video/upload/v1726932262/n91xtzqavpdoro79lnza.mp4" controls="true"}

### 调试 Edge 模板

您可以像调试 TypeScript 编写的应用程序代码一样调试 Edge 模板。但是，对于 Edge，您不能使用 VSCode 提供的断点。相反，您必须使用 `@debugger` 标签来定义代码内断点。

:::note

调试器将显示 Edge 模板的编译输出。

:::

```edge
@debugger
```

## 转储并终止

转储并终止（称为 `dd`）类似于最受欢迎的调试技术 `console.log`。但是，`dd` 助手将通过抛出异常来停止执行，并在浏览器或终端内显示输出。

当您在 HTTP 请求期间使用 `dd` 助手时，输出将呈现为 HTML 文档。否则，输出将显示在终端内。

```ts
// title: start/routes.ts
import User from '#models/user'
import router from '@adonisjs/core/services/router'
// highlight-start
import { dd } from '@adonisjs/core/services/dumper'
// highlight-end

router.get('/users', async () => {
  const users = await User.all()
  // highlight-start
  /**
   * 访问 "/users" 端点以查看转储的值
   */
  dd(users)
  // highlight-end
  return users
})
```

`dd` 的输出与您使用 `console.log` 时看到的略有不同。

- 您可以看到转储值的源代码位置。
- 您可以查看类的静态属性和对象的原型属性。
- 默认情况下，显示嵌套值最多 10 层深。
- 支持 HTML 输出的多个主题。您可以选择 `nightOwl`、`catppuccin` 和 `minLight` 之间。

![控制台输出](./dd-cli.png)

![浏览器输出](./browser-dd.png)

### 用于调试的 Edge 助手

您可以通过 `@dd` 标签在 Edge 模板内使用 `dd` 助手。此外，您可以使用 `@dump` 助手，它不会抛出异常并继续渲染模板的其余部分。

```edge
{{-- 转储模板状态并终止 --}}
@dd(state)

{{-- 转储模板状态并继续渲染 --}}
@dump(state)
```

使用 `@dump` 助手时，请确保页面上有一个名为 "dumper" 的 [EdgeJS Stack](https://edgejs.dev/docs/stacks)。`@dump` 助手使用的脚本和样式将写入此堆栈，以包含在最终的 HTML 输出中。

```edge
<!DOCTYPE html>
  <html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    @stack('dumper')
  </head>
  <body>
    @dump(state)
  </body>
</html>
```

### 转储器设置

您可以在 `config/app.ts` 文件内配置转储器设置。此文件应导出一个 `dumper` 配置对象，如下所示。

```ts
// title: config/app.ts
/**
 * "dd" 助手使用的全局配置。您可以
 * 分别为 "console" 和 "html" 打印机配置设置。
 */
export const dumper = dumperConfig({
  /**
   * 控制台打印机的设置
   */
  console: {
    depth: 10,

    /**
     * 不应进一步展开的对象。该
     * 数组接受对象构造函数
     * 名称数组。
     */
    collapse: ['DateTime', 'Date'],
    inspectStaticMembers: true,
  },

  /**
   * HTML 打印机的设置
   */
  html: {
    depth: 10,
    inspectStaticMembers: true,
  },
})
```

<dl>

<dt> showHidden </dt>
<dd>

当设置为 `true` 时，将处理对象的不可枚举属性。**默认值：`false`**

</dd>

<dt> depth </dt>
<dd>

停止解析嵌套值的深度。该深度在所有树状数据结构之间共享。例如，对象、数组、映射和集合。**默认值：`5`**

</dd>

<dt> inspectObjectPrototype </dt>
<dd>

检查对象的原型属性。默认情况下包含原型的不可枚举属性。**默认值：`'unless-plain-object'`**。

- 当设置为 `true` 时，将为所有对象处理原型属性。
- 当设置为 `false` 时，从不处理原型属性。
- 当设置为 `'unless-plain-object'` 时，将处理类实例的原型属性。

</dd>

<dt> inspectArrayPrototype </dt>
<dd>

检查数组的原型属性。**默认值：`false`**。

</dd>

<dt> inspectStaticMembers </dt>
<dd>

检查类的静态成员。尽管函数和类在技术上是相同的，但此配置仅适用于使用 `[class]` 关键字定义的函数。**默认值：`false`**。

</dd>

<dt> maxArrayLength </dt>
<dd>

要处理的 `数组`、`映射` 和 `集合` 的最大成员数。**默认值：`100`**。

</dd>

<dt> maxStringLength </dt>
<dd>

字符串要显示的最大字符数。**默认值：`1000`**。

</dd>

<dt> collapse </dt>
<dd>

不应进一步检查的对象构造函数名称数组。

</dd>

</dl>

## 框架调试日志

您可以使用 `NODE_DEBUG` 环境变量查看框架调试日志。`NODE_DEBUG` 标志由 Node.js 运行时支持，您可以使用它来查看一个或多个模块的日志，使用模块名称。

例如，您可以使用 `NODE_DEBUG="adonisjs:*"` 值查看所有 AdonisJS 包的日志。

```sh
NODE_DEBUG="adonisjs:*" node ace serve --hmr
```

同样，您可以使用 `NODE_DEBUG` 环境变量查看来自 Node.js 本机模块（如 `fs`、`net`、`module` 等）的日志。

```sh
NODE_DEBUG="adonisjs:*,net,fs" node ace serve --hmr
```

<!-- ## 观察活动
观察应用程序不同部分执行的操作可以提供一些有意义的见解，并帮助您调试和理解框架的内部工作原理。

### Lucid 查询
您可以通过在 `config/database.ts` 文件中启用 `debug` 标志来调试 Lucid 查询。此外，您必须在同一文件中启用 `prettyPrintDebugQueries` 以在终端中打印 SQL 查询。

```ts
// title: config/database.ts
const dbConfig = defineConfig({
  connection: 'sqlite',
  // insert-start
  prettyPrintDebugQueries: true,
  // insert-end
  connections: {
    sqlite: {
      client: 'better-sqlite3',
      connection: {
        filename: app.tmpPath('db.sqlite3'),
      },
      // insert-start
      debug: true,
      // insert-end
      useNullAsDefault: true,
      migrations: {
        naturalSort: true,
        paths: ['database/migrations'],
      },
    },
  },
})
```

![](./debug-sql-queries.png) -->

<!-- ### 外发电子邮件

### HTTP 请求 -->
