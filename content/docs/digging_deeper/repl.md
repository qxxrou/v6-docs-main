---
summary: AdonisJS提供了一个应用感知的REPL，用于从命令行与您的应用交互。
---

# REPL

与[Node.js REPL](https://nodejs.org/api/repl.html)类似，AdonisJS提供了一个应用感知的REPL，用于从命令行与您的应用交互。您可以使用`node ace repl`命令启动REPL会话。

```sh
node ace repl
```

![](../ace/ace_repl.png)

除了标准的Node.js REPL功能外，AdonisJS还提供了以下特性：

- 导入和执行TypeScript文件。
- 导入容器服务（如`router`、`helpers`、`hash`服务等）的简写方法。
- 使用[IoC容器](../concepts/dependency_injection.md#constructing-a-tree-of-dependencies)创建类实例的简写方法。
- 可扩展的API，用于添加自定义方法和REPL命令。

## 与REPL交互

启动REPL会话后，您将看到一个交互式提示符，可以在其中编写有效的JavaScript代码并按回车键执行。代码的输出将打印在下一行。

如果您想输入多行代码，可以通过输入`.editor`命令进入编辑器模式。按`Ctrl+D`执行多行语句或按`Ctrl+C`取消并退出编辑器模式。

```sh
> (js) .editor
# // 进入编辑器模式 (Ctrl+D完成，Ctrl+C取消)
```

### 访问上次执行命令的结果

如果您忘记将语句的值分配给变量，可以使用`_`变量访问它。例如：

```sh
> (js) helpers.string.generateRandom(32)
# 'Z3y8QQ4HFpYSc39O2UiazwPeKYdydZ6M'
> (js) _
# 'Z3y8QQ4HFpYSc39O2UiazwPeKYdydZ6M'
> (js) _.length
# 32
> (js)
```

### 访问上次执行命令引发的错误

您可以使用`_error`变量访问上一个命令引发的异常。例如：

```sh
> (js) helpers.string.generateRandom()
> (js) _error.message
# 'The value of "size" is out of range. It must be >= 0 && <= 2147483647. Received NaN'
```

### 在历史记录中搜索

REPL历史记录保存在用户主目录中的`.adonisjs_v6_repl_history`文件中。

您可以按向上箭头`↑`键循环浏览历史命令，或按`Ctrl+R`在历史记录中搜索。

### 退出REPL会话

您可以通过输入`.exit`或按两次`Ctrl+C`退出REPL会话。AdonisJS将在关闭REPL会话前执行优雅关闭。

此外，如果您修改了代码库，必须退出并重新启动REPL会话才能获取新的更改。

## 导入模块

Node.js不允许在REPL会话中使用`import`语句。因此，您必须使用动态`import`函数并将输出分配给变量。例如：

```ts
const { default: User } = await import('#models/user')
```

您可以使用`importDefault`方法访问默认导出，而无需解构导出。

```ts
const User = await importDefault('#models/user')
```

## 辅助方法

辅助方法是您可以执行以执行特定操作的快捷函数。您可以使用`.ls`命令查看可用方法列表。

```sh
> (js) .ls

# 全局方法:
importDefault         返回模块的默认导出
make                  使用"container.make"方法创建类实例
loadApp               在REPL上下文中加载"app"服务
loadEncryption        在REPL上下文中加载"encryption"服务
loadHash              在REPL上下文中加载"hash"服务
loadRouter            在REPL上下文中加载"router"服务
loadConfig            在REPL上下文中加载"config"服务
loadTestUtils         在REPL上下文中加载"testUtils"服务
loadHelpers           在REPL上下文中加载"helpers"模块
clear                 从REPL上下文中清除属性
p                     将函数Promise化。类似于Node.js的"util.promisify"
```

## 向REPL添加自定义方法

您可以使用`repl.addMethod`向REPL添加自定义方法。该方法接受名称作为第一个参数，实现回调作为第二个参数。

为了演示，让我们创建一个[预加载文件](../concepts/adonisrc_file.md#preloads)并定义一个方法，用于从`./app/models`目录导入所有模型。

```sh
node ace make:preload repl -e=repl
```

```ts
// title: start/repl.ts
import app from '@adonisjs/core/services/app'
import repl from '@adonisjs/core/services/repl'
import { fsImportAll } from '@adonisjs/core/helpers'

repl.addMethod('loadModels', async () => {
  const models = await fsImportAll(app.makePath('app/models'))
  repl.server!.context.models = models

  repl.notify('已导入模型。您可以使用"models"属性访问它们')
  repl.server!.displayPrompt()
})
```

您可以将以下选项作为第三个参数传递给`repl.addMethod`方法：

- `description`：在帮助输出中显示的人类可读描述。
- `usage`：定义方法使用代码片段。如果未设置，将使用方法名称。

完成后，您可以重新启动REPL会话并执行`loadModels`方法来导入所有模型。

```sh
node ace repl

# 键入".ls"查看可用的上下文方法/属性
> (js) await loadModels()
```
