---
summary: 学习如何在 AdonisJS 中使用 Japa（我们的内置测试框架）编写和运行测试。
---

# 测试

AdonisJS 内置了对编写测试的支持。你无需安装额外的包或配置应用程序以准备测试 - 所有繁重的工作都已经完成。

你可以使用以下 ace 命令运行应用程序测试：

```sh
node ace test
```

测试存储在 `tests` 目录中，我们按类型进一步组织测试。例如，功能测试存储在 `tests/functional` 目录中，单元测试存储在 `tests/unit` 目录中。

功能测试指的是外部测试，在这种测试中，你将对应用程序进行真实的 HTTP 请求以测试给定流程或端点的功能。例如，你可能有一系列用于创建用户的功能测试。

某些社区可能将功能测试称为特性测试或端到端测试。AdonisJS 对你如何称呼它们很灵活。我们决定采用术语 **功能测试**。

## 配置测试运行器

AdonisJS 使用 [Japa](https://japa.dev/docs) 来编写和运行测试。因此，我们建议阅读 Japa 文档以更好地了解其 API 和配置选项。

### 测试套件

测试套件在 `adonisrc.ts` 文件中定义。默认情况下，我们注册了 `functional` 和 `unit` 测试套件。如果需要，你可以删除现有套件并从头开始。

```ts
{
  tests: {
    suites: [
      {
        name: 'functional',
        files: ['tests/functional/**/*.spec.(js|ts)'],
      },
      {
        name: 'unit',
        files: ['tests/unit/**/*.spec.(js|ts)'],
      },
    ]
  }
}
```

- 套件结合了套件的唯一名称和文件的 glob 模式。
- 当你运行特定套件的测试时，仅导入与该套件相关的文件。

你可以在 `tests/bootstrap.ts` 文件中定义的 `configureSuite` 钩子中在运行时配置套件。例如，在运行功能测试时，你可以注册套件级别的钩子来启动 HTTP 服务器。

```ts
export const configureSuite: Config['configureSuite'] = (suite) => {
  if (['browser', 'functional', 'e2e'].includes(suite.name)) {
    return suite.setup(() => testUtils.httpServer().start())
  }
}
```

### 运行器钩子

运行器钩子是在所有测试之前和之后可以运行的全局操作。钩子使用 `tests/bootstrap.ts` 文件中的 `runnerHooks` 属性定义。

```ts
export const runnerHooks: Required<Pick<Config, 'setup' | 'teardown'>> = {
  setup: [
    () => {
      console.log('在所有测试之前运行')
    },
  ],
  teardown: [
    () => {
      console.log('在所有测试之后运行')
    },
  ],
}
```

### 插件

Japa 有一个插件系统，你可以使用它来扩展其功能。插件在 `tests/bootstrap.ts` 文件中注册。

另请参阅：[创建 Japa 插件](https://japa.dev/docs/creating-plugins)

```ts
export const plugins: Config['plugins'] = [assert(), pluginAdonisJS(app)]
```

### 报告器

报告器用于在测试运行时报告/显示测试进度。报告器在 `tests/bootstrap.ts` 文件中注册。

另请参阅：[创建 Japa 报告器](https://japa.dev/docs/creating-reporters)

```ts
export const reporters: Config['reporters'] = {
  activated: ['spec'],
}
```

## 创建测试

你可以使用 `make:test` 命令创建新测试。该命令需要套件的名称来创建测试文件。

另请参阅：[Make test 命令](../references/commands.md#maketest)

```sh
node ace make:test posts/create --suite=functional
```

文件将使用 `files` glob 属性配置的目录中创建。

## 编写测试

测试使用从 `@japa/runner` 包导入的 `test` 方法定义。测试接受标题作为第一个参数，实现回调作为第二个参数。

在以下示例中，我们创建一个新用户账户并使用 [`assert`](https://japa.dev/docs/plugins/assert) 对象确保密码正确哈希。

```ts
import { test } from '@japa/runner'

import User from '#models/User'
import hash from '@adonisjs/core/services/hash'

test('在创建新用户时哈希用户密码', async ({ assert }) => {
  const user = new User()
  user.password = 'secret'

  await user.save()

  assert.isTrue(hash.isValidHash(user.password))
  assert.isTrue(await hash.verify(user.password, 'secret'))
})
```

### 使用测试组

测试组使用 `test.group` 方法创建。组为你的测试添加结构，并允许你在测试周围运行[生命周期钩子](https://japa.dev/docs/lifecycle-hooks)。

继续前面的示例，让我们将密码哈希测试移到组内。

```ts
import { test } from '@japa/runner'

import User from '#models/User'
import hash from '@adonisjs/core/services/hash'

// highlight-start
test.group('创建用户', () => {
  // highlight-end
  test('哈希用户密码', async ({ assert }) => {
    const user = new User()
    user.password = 'secret'

    await user.save()

    assert.isTrue(hash.isValidHash(user.password))
    assert.isTrue(await hash.verify(user.password, 'secret'))
  })
  // highlight-start
})
// highlight-end
```

如果你注意到，我们从测试标题中删除了 **"在创建新用户时"** 片段。这是因为组标题阐明了此组下的所有测试都限定于 **创建新用户**。

### 生命周期钩子

生命周期钩子用于在测试周围执行操作。你可以使用 `group` 对象定义钩子。

另请参阅 - [Japa 生命周期钩子文档](https://japa.dev/docs/lifecycle-hooks)

```ts
test.group('创建用户', (group) => {
  // highlight-start
  group.each.setup(async () => {
    console.log('在每个测试之前运行')
  })

  group.each.teardown(async () => {
    console.log('在每个测试之后运行')
  })

  group.setup(async () => {
    console.log('在所有测试之前运行一次')
  })

  group.teardown(async () => {
    console.log('在所有测试之后运行一次')
  })
  // highlight-end

  test('哈希用户密码', async ({ assert }) => {
    const user = new User()
    user.password = 'secret'

    await user.save()

    assert.isTrue(hash.isValidHash(user.password))
    assert.isTrue(await hash.verify(user.password, 'secret'))
  })
})
```

### 后续步骤

现在你已经了解了创建和编写测试的基础知识。我们建议你在 Japa 文档中探索以下主题。

- [探索 `test` 函数 API](https://japa.dev/docs/underlying-test-class)
- [学习如何有效地测试异步代码](https://japa.dev/docs/testing-async-code)
- [使用数据集避免重复测试](https://japa.dev/docs/datasets)

## 运行测试

你可以使用 `test` 命令运行测试。默认情况下，执行所有套件的测试。但是，你可以通过传递名称来运行特定套件的测试。

```sh
node ace test
```

```sh
node ace test functional
node ace test unit
```

### 监视文件更改并重新运行测试

你可以使用 `--watch` 命令来监视文件系统并重新运行测试。如果测试文件被更改，则将运行更改文件内的测试。否则，将重新运行所有测试。

```sh
node ace test --watch
```

### 过滤测试

在运行测试时，你可以使用命令行标志应用过滤器。以下是可用选项列表。

另请参阅：[Japa 过滤测试指南](https://japa.dev/docs/filtering-tests)

:::tip

**使用 VSCode？** 使用 [Japa 扩展](https://marketplace.visualstudio.com/items?itemName=jripouteau.japa-vscode) 使用键盘快捷键或活动侧边栏在代码编辑器中运行选定的测试。

:::

| 标志         | 描述                                                                                                                         |
| ------------ | ---------------------------------------------------------------------------------------------------------------------------- |
| `--tests`    | 按测试标题过滤测试。此过滤器与确切的测试标题匹配。                                                                           |
| `--files`    | 按测试文件名的子集过滤测试。匹配针对没有 `.spec.ts` 的文件名末尾执行。你可以使用通配符表达式运行完整文件夹的测试。`folder/*` |
| `--groups`   | 按组名称过滤测试。此过滤器与确切的组名称匹配。                                                                               |
| `--tags`     | 按标签过滤测试。你可以在标签名称前加上波浪号 `~` 来忽略具有给定标签的测试                                                    |
| `--matchAll` | 默认情况下，Japa 将运行与任何提到的标签匹配的测试。如果你希望所有标签都匹配，请使用 `--matchAll` 标志                        |

### 强制退出测试

Japa 等待进程在完成所有测试后正常关闭。优雅关闭过程意味着退出所有长期连接并清空 Node.js 事件循环。

如果需要，你可以使用 `--force-exit` 标志强制 Japa 退出进程而不等待优雅关闭。

```sh
node ace test --force-exit
```

### 重试测试

你可以使用 `--retries` 标志多次重试失败的测试。该标志将应用于在测试级别没有明确定义重试计数的所有测试。

```sh
# 重试失败的测试 2 次
node ace test --retries=2
```

### 从上次运行中运行失败的测试

你可以使用 `--failed` 命令行标志重新运行上次运行中失败的测试。

```sh
node ace test --failed
```

### 在报告器之间切换

Japa 允许你在配置文件中注册多个测试报告器，但默认情况下不激活它们。你可以在配置文件内激活报告器，或使用 `--reporter` 命令行标志。

```sh
# 激活规范报告器
node ace test --reporter=spec

# 激活规范和 json 报告器
node ace test --reporter=spec,json
```

你也可以在配置文件中激活报告器。

```ts
export const reporters: Config['reporters'] = {
  activated: ['spec', 'json'],
}
```

### 向 Node.js 命令行传递选项

`test` 命令将测试作为子进程运行（`bin/test.ts` 文件）。如果你想向子进程传递 [node 参数](https://nodejs.org/api/cli.html#options)，你可以在命令名称之前定义它们。

```sh
node ace --no-warnings --trace-exit test
```

## 环境变量

你可以使用 `.env.test` 文件来定义测试期间所需的环境变量。`.env.test` 内的值优先于 `.env` 文件内的值。

测试期间的 `SESSION_DRIVER` 必须设置为 `memory`。

```dotenv
// title: .env.test
SESSION_DRIVER=memory
```
