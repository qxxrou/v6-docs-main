---
 summary: 了解如何使用宏和 getters 扩展 AdonisJS 框架。
---



# 扩展框架

AdonisJS 的架构使得扩展框架变得非常容易。我们使用框架的核心 API 来构建第一方包的生态系统。

在本指南中，我们将探索可以用来通过包或应用程序代码库扩展框架的不同 API。

## Macros 和 getters

Macros 和 getters 提供了一种将属性添加到类原型的 API。您可以将它们视为 `Object.defineProperty` 的语法糖。在内部，我们使用 [macroable](https://github.com/poppinss/macroable) 包，您可以参考其 README 以获取深入的技术说明。

由于 macros 和 getters 是在运行时添加的，您必须使用 [声明合并](https://www.typescriptlang.org/docs/handbook/declaration-merging.html) 向 TypeScript 提供所添加属性的类型信息。

您可以将添加 macros 的代码写在一个专用文件（如 `extensions.ts`）中，并在服务提供者的 `boot` 方法中导入它。

```ts
// 标题: providers/app_provider.ts
export default class AppProvider {
  async boot() {
    await import('../src/extensions.js')
  }
}
```

在下面的示例中，我们将 wantsJSON 方法添加到 Request 类中，同时定义其类型。

```ts
// 标题: src/extensions.ts
import { Request } from '@adonisjs/core/http'

Request.macro('wantsJSON', function (this: Request) {
  const firstType = this.types()[0]
  if (!firstType) {
    return false
  }

  return firstType.includes('/json') || firstType.includes('+json')
})
```

````ts

// 标题: src/extensions.ts
declare module '@adonisjs/core/http' {
  interface Request {
    wantsJSON(): boolean
  }
}```
````

- 在 `declare module` 调用期间，模块路径必须与您用于导入类的路径相同。
- `interface` 名称必须与您添加 macro 或 getter 的类名称相同。

### Getters

Getters 是添加到类中的延迟求值属性。您可以使用 `Class.getter` 方法添加 getter。第一个参数是 getter 名称，第二个参数是计算属性值的回调函数。

Getter 回调不能是异步的，因为 JavaScript 中的 [getters](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/get) 不能是异步的。

```ts
import { Request } from '@adonisjs/core/http'

Request.getter('hasRequestId', function (this: Request) {
  return this.header('x-request-id')
})

// 您可以如下使用该属性。
if (ctx.request.hasRequestId) {
}
```

Getters 可以是单例的，这意味着计算 getter 值的函数将被调用一次，返回值将为类的一个实例缓存。

```ts
const isSingleton = true

Request.getter(
  'hasRequestId',
  function (this: Request) {
    return this.header('x-request-id')
  },
  isSingleton,
)
```

Getters 可以是单例的，这意味着计算 getter 值的函数将被调用一次，返回值将为类的一个实例缓存。

```ts
const isSingleton = true

Request.getter(
  'hasRequestId',
  function (this: Request) {
    return this.header('x-request-id')
  },
  isSingleton,
)
```

## 可扩展的类

以下是可以通过 Macros 和 getters 扩展的类列表。

类名 导入路径
| Class | Import path |
|------------------------------------------------------------------------------------------------|-----------------------------|
| [Application](https://github.com/adonisjs/application/blob/main/src/application.ts) | `@adonisjs/core/app` |
| [Request](https://github.com/adonisjs/http-server/blob/main/src/request.ts) | `@adonisjs/core/http` |
| [Response](https://github.com/adonisjs/http-server/blob/main/src/response.ts) | `@adonisjs/core/http` |
| [HttpContext](https://github.com/adonisjs/http-server/blob/main/src/http_context/main.ts) | `@adonisjs/core/http` |
| [Route](https://github.com/adonisjs/http-server/blob/main/src/router/route.ts) | `@adonisjs/core/http` |
| [RouteGroup](https://github.com/adonisjs/http-server/blob/main/src/router/group.ts) | `@adonisjs/core/http` |
| [RouteResource](https://github.com/adonisjs/http-server/blob/main/src/router/resource.ts) | `@adonisjs/core/http` |
| [BriskRoute](https://github.com/adonisjs/http-server/blob/main/src/router/brisk.ts) | `@adonisjs/core/http` |
| [ExceptionHandler](https://github.com/adonisjs/http-server/blob/main/src/exception_handler.ts) | `@adonisjs/core/http` |
| [MultipartFile](https://github.com/adonisjs/bodyparser/blob/main/src/multipart/file.ts) | `@adonisjs/core/bodyparser` |

## 扩展模块

大多数 AdonisJS 模块都提供了可扩展的 API 来注册自定义实现。以下是相同的汇总列表。

- [创建哈希驱动程序](../security/hashing.md#creating-a-custom-hash-driver)
- [创建会话驱动程序](../basics/session.md#creating-a-custom-session-store)
- [创建社交认证驱动程序](../authentication/social_authentication.md#creating-a-custom-social-driver)
- [扩展 REPL](../digging_deeper/repl.md#adding-custom-methods-to-repl)
- [创建国际化翻译加载器](../digging_deeper/i18n.md#creating-a-custom-translation-loader)
- [创建国际化翻译格式化器](../digging_deeper/i18n.md#creating-a-custom-translation-formatter)
