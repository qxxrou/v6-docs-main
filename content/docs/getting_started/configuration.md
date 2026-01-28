---
summary: 学习如何在 AdonisJS 中读取和更新配置值。
---

# 配置

你的 AdonisJS 应用程序的配置文件存储在 `config` 目录中。一个全新的 AdonisJS 应用程序附带了一些预先存在的文件，这些文件被框架核心和已安装的包使用。

请随意在 `config` 目录中创建应用程序所需的额外文件。

:::note

我们推荐使用[环境变量](./environment_variables.md)来存储机密信息和环境特定的配置。

:::

## 导入配置文件

你可以在应用程序代码库中使用标准的 JavaScript `import` 语句导入配置文件。例如：

```ts
import { appKey } from '#config/app'
```

```ts
import databaseConfig from '#config/database'
```

当你导入配置文件时，你可以访问导出的值。在大多数情况下，这些导出是 [`ConfigProvider`](../concepts/config_providers.md) 实例，因此不建议直接使用它们的值。相反，请从[解析后的配置](../concepts/config_providers.md#how-do-i-access-the-resolved-config)中读取值。

## 使用配置服务

配置服务提供了另一种用于读取配置值的 API。在下面的示例中，我们使用配置服务读取存储在 `config/app.ts` 文件中的 `appKey` 值。

```ts
import config from '@adonisjs/core/services/config'

config.get('app.appKey')
config.get('app.http.cookie') // 读取嵌套值
```

`config.get` 方法接受一个点分隔的键，并按如下方式解析：

- 第一部分是你想要读取值的文件名。即 `app.ts` 文件。
- 其余字符串片段是你想要从导出值中访问的键。即本例中的 `appKey`。

### 配置服务与直接导入配置文件

使用配置服务而不是直接导入配置文件没有直接的好处。然而，在外部包和 Edge 模板中读取配置时，配置服务是唯一的选择。

### 在外部包中读取配置

如果你正在创建第三方包，你不应该直接从用户应用程序导入配置文件，因为这会使你的包与宿主应用程序的文件夹结构紧密耦合。

相反，你应该在服务提供商内部使用配置服务来访问配置值。例如：

```ts
import { ApplicationService } from '@adonisjs/core/types'

export default class DriveServiceProvider {
  constructor(protected app: ApplicationService) {}
  
  register() {
    this.app.container.singleton('drive', () => {
      // highlight-start
      const driveConfig = this.app.config.get('drive')
      return new DriveManager(driveConfig)
      // highlight-end
    })
  }
}
```

### 在 Edge 模板中读取配置

你可以在 Edge 模板中使用 `config` 全局方法访问配置值。

```edge
<a href="{{ config('app.appUrl') }}"> 首页 </a>
```

你可以使用 `config.has` 方法来检查给定键是否存在配置值。如果值为 `undefined`，该方法返回 `false`。

```edge
@if(config.has('app.appUrl'))
  <a href="{{ config('app.appUrl') }}"> 首页 </a>
@else
  <a href="/"> 首页 </a>
@end
```

## 更改配置位置

你可以通过修改 [`adonisrc.ts`](../concepts/adonisrc_file.md) 文件来更新配置目录的位置。更改后，配置文件将从新位置导入。

```ts
directories: {
  config: './configurations'
}
```

确保更新 `package.json` 文件中的导入别名。

```json
{
  "imports": {
    "#config/*": "./configurations/*.js"
  }
}
```

## 配置文件限制

存储在 `config` 目录中的配置文件在应用程序的引导阶段被导入。因此，配置文件不能依赖于应用程序代码。

例如，如果你尝试在 `config/app.ts` 文件中导入和使用路由器服务，应用程序将无法启动。这是因为路由器服务在应用程序处于 `booted` 状态之前不会被配置。

从根本上说，这种限制对你的代码库产生了积极影响，因为应用程序代码应该依赖于配置，而不是相反。

## 运行时更新配置

你可以使用配置服务在运行时改变配置值。`config.set` 在内存中更新值，不会对磁盘上的文件进行任何更改。

:::note

配置值是为整个应用程序改变的，而不仅仅是针对单个 HTTP 请求。这是因为 Node.js 不是线程化运行时，Node.js 中的内存在多个 HTTP 请求之间共享。

:::

```ts
import env from '#start/env'
import config from '@adonisjs/core/services/config'

const HOST = env.get('HOST')
const PORT = env.get('PORT')

config.set('app.appUrl', `http://${HOST}:${PORT}`)
```
