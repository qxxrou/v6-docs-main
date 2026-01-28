---
summary: Ace 是 AdonisJS 使用的命令行框架，用于创建和运行控制台命令。
---

# 介绍

Ace 是 AdonisJS 使用的命令行框架，用于创建和运行控制台命令。Ace 的入口点文件存储在项目的根目录中，您可以按以下方式执行它。

```sh
node ace
```

由于 `node` 二进制文件不能直接运行 TypeScript 源代码，我们必须将 ace 文件保持为纯 JavaScript 并使用 `.js` 扩展名。

在底层，`ace.js` 文件将 TS Node 注册为 [ESM 模块加载器钩子](https://nodejs.org/api/module.html#customization-hooks) 来执行 TypeScript 代码并导入 `bin/console.ts` 文件。

## 帮助和列表命令

您可以通过运行 ace 入口点文件而不带任何参数或使用 `list` 命令来查看可用命令列表。

```sh
node ace

# 同上
node ace list
```

![](./ace_help_screen.jpeg)

您可以通过输入命令名称和 `--help` 标志来查看单个命令的帮助。

```sh
node ace make:controller --help
```

:::note

帮助屏幕的输出格式符合 [docopt](http://docopt.org/) 标准。

:::

## 启用/禁用颜色

Ace 会检测它运行的 CLI 环境，如果终端不支持颜色，则会禁用彩色输出。但是，您可以使用 `--ansi` 标志手动启用或禁用颜色。

```sh
# 禁用颜色
node ace list --no-ansi

# 强制启用颜色
node ace list --ansi
```

## 创建命令别名

命令别名为常用命令定义别名提供了便利层。例如，如果您经常创建单数资源控制器，您可以在 `adonisrc.ts` 文件中为其创建别名。

```ts
{
  commandsAliases: {
    resource: 'make:controller --resource --singular'
  }
}
```

一旦定义了别名，您就可以使用别名来运行命令。

```sh
node ace resource admin
```

### 别名扩展如何工作？

- 每次运行命令时，Ace 都会检查 `commandsAliases` 对象中的别名。
- 如果存在别名，将使用第一个段（空格之前）来查找命令。
- 如果命令存在，别名值中的其余段将附加到命令名称。

  例如，如果您运行以下命令

  ```sh
  node ace resource admin --help
  ```

  它将被扩展为

  ```sh
  make:controller --resource --singular admin --help
  ```

## 以编程方式运行命令

您可以使用 `ace` 服务来执行命令。ace 服务在应用程序启动后可用。

`ace.exec` 方法接受命令名称作为第一个参数，命令行参数数组作为第二个参数。例如：

```ts
import ace from '@adonisjs/core/services/ace'

const command = await ace.exec('make:controller', ['user', '--resource'])

console.log(command.exitCode)
console.log(command.result)
console.log(command.error)
```

您可以使用 `ace.hasCommand` 方法在执行前检查命令是否存在。

```ts
import ace from '@adonisjs/core/services/ace'

/**
 * boot 方法将加载命令（如果尚未加载）
 */
await ace.boot()

if (ace.hasCommand('make:controller')) {
  await ace.exec('make:controller', ['user', '--resource'])
}
```
