---
summary: Learn about the HTTP context in AdonisJS and how to access it from route handlers, middleware, and exception handlers.
---

# HTTP 上下文

每次 HTTP 请求都会生成一个 [HTTP Context 类](https://github.com/adonisjs/http-server/blob/main/src/http_context/main.ts) 的新实例，并将其传递给路由处理器、中间件和异常处理器。

HTTP Context 包含了与 HTTP 请求相关的所有信息。例如：

- 您可以使用 [ctx.request](../basics/request.md) 属性访问请求体、头部和查询参数。
- 您可以使用 [ctx.response](../basics/response.md) 属性响应 HTTP 请求。
- 使用 [ctx.auth](../authentication/introduction.md) 属性访问登录用户。
- 使用 [ctx.bouncer](../security/authorization.md) 属性授权用户操作。
- 等等。

简而言之，context 是一个特定于请求的存储，保存着当前请求的所有信息。

## 获取对 HTTP 上下文的访问

HTTP 上下文通过引用传递给路由处理器、中间件和异常处理器，您可以按如下方式访问它。

### 路由处理器

[路由器处理器](../basics/routing.md) 将 HTTP 上下文作为第一个参数接收。

```ts
import router from '@adonisjs/core/services/router'

router.get('/', (ctx) => {
  console.log(ctx.inspect())
})
```

```ts
// title: Destructure properties
import router from '@adonisjs/core/services/router'

router.get('/', ({ request, response }) => {
  console.log(request.url())
  console.log(request.headers())
  console.log(request.qs())
  console.log(request.body())

  response.send('hello world')
  response.send({ hello: 'world' })
})
```

### 控制器方法

[控制器方法](../basics/controllers.md) （类似于路由器处理器）将 HTTP 上下文作为第一个参数接收。

```ts
import { HttpContext } from '@adonisjs/core/http'

export default class HomeController {
  async index({ request, response }: HttpContext) {}
}
```

### 中间件 class

[middleware class](../basics/middleware.md) 的 `handle` 方法将 HTTP 上下文作为第一个参数接收。

```ts
import { HttpContext } from '@adonisjs/core/http'

export default class AuthMiddleware {
  async handle({ request, response }: HttpContext) {}
}
```

### 异常处理器类

[全局异常处理器 ](../basics/exception_handling.md) 类的 `handle` 和 `report` 方法将 HTTP 上下文作为第二个参数接收。第一个参数是 `error` 属性。

```ts
import { HttpContext, HttpExceptionHandler } from '@adonisjs/core/http'

export default class ExceptionHandler extends HttpExceptionHandler {
  async handle(error: unknown, ctx: HttpContext) {
    return super.handle(error, ctx)
  }

  async report(error: unknown, ctx: HttpContext) {
    return super.report(error, ctx)
  }
}
```

## 使用依赖注入注入 HTTP 上下文

如果您在应用程序中使用依赖注入，您可以通过类型提示 `HttpContext` 类来将 HTTP 上下文注入到类或方法中。

:::warning

确保在 `#middleware/container_bindings_middleware` 文件中注册了 `start/kernel.ts` 中间件。此中间件对于从容器解析特定于请求的值（即 HttpContext 类）是必需的。

:::

另请参阅: [IoC 容器指南](../concepts/dependency_injection.md)

```ts
// title: app/services/user_service.ts
import { inject } from '@adonisjs/core'
import { HttpContext } from '@adonisjs/core/http'

@inject()
export default class UserService {
  constructor(protected ctx: HttpContext) {}

  all() {
    // method implementation
  }
}
```

为了使自动依赖解析正常工作，您必须将 `UserService` 注入到控制器中。记住，控制器方法的第一个参数始终是上下文，其余的将使用 IoC 容器注入。

```ts
import { inject } from '@adonisjs/core'
import { HttpContext } from '@adonisjs/core/http'
import UserService from '#services/user_service'

export default class UsersController {
  @inject()
  index(ctx: HttpContext, userService: UserService) {
    return userService.all()
  }
}
```

就这样！现在 `UserService` 将自动接收到正在进行的 HTTP 请求的实例。您也可以对嵌套依赖重复相同的过程。

## 在应用程序中的任何地方访问 HTTP 上下文

依赖注入是一种接受 HTTP 上下文作为类构造函数或方法依赖并依靠容器为您解析它的方法。

However, it is not a hard requirement to restructure your application and use Dependency injection everywhere. You can also access the HTTP context from anywhere inside your application using the [Async local storage](https://nodejs.org/dist/latest-v21.x/docs/api/async_context.html#class-asynclocalstorage) provided by Node.js.

但是，不需要重构您的应用程序并在各处使用依赖注入。您还可以使用 Node.js 提供的 [异步本地存储](./async_local_storage.md) 从应用程序中的任何位置访问 HTTP 上下文。

在下面的示例中，`UserService` 类使用 `HttpContext.getOrFail` 方法获取当前请求的 HTTP 上下文实例。

```ts
// title: app/services/user_service.ts
import { HttpContext } from '@adonisjs/core/http'

export default class UserService {
  all() {
    const ctx = HttpContext.getOrFail()
    console.log(ctx.request.url())
  }
}
```

以下代码块显示了在 `UserService` 中使用 `UsersController` class.

```ts
import { HttpContext } from '@adonisjs/core/http'
import UserService from '#services/user_service'

export default class UsersController {
  index(ctx: HttpContext) {
    const userService = new UserService()
    return userService.all()
  }
}
```

## HTTP 上下文属性

以下是您可以通过 HTTP 上下文访问的属性列表。当您安装新包时，它们可能会向上下文添加其他属性。

<dl>
<dt>

ctx.request

</dt>

<dd>

[HTTP Request 类 ](../basics/request.md) 实例的引用.

</dd>

<dt>

ctx.response

</dt>

<dd>

[HTTP Response 类 ](../basics/response.md) 实例的引用 .

</dd>

<dt>

ctx.logger

</dt>

<dd>

为给定 HTTP 请求创建的 [logger](../digging_deeper/logger.md) 实例的引用。

</dd>

<dt>

ctx.route

</dt>

<dd>

当前 HTTP 请求匹配的路由 `route` 属性是一个 [StoreRouteNode](https://github.com/adonisjs/http-server/blob/main/src/types/route.ts#L69) 类型的对象

</dd>

<dt>

ctx.params

</dt>

<dd>

路由参数对象

</dd>

<dt>

ctx.subdomains

</dt>

<dd>

路由子域名的对象。仅当路由是动态子域的一部分时存在

</dd>

<dt>

ctx.session

</dt>

<dd>

为当前 HTTP 请求创建的 [Session](../basics/session.md) 实例的引用。

</dd>

<dt>

ctx.auth

</dt>

<dd>

[认证器类](https://github.com/adonisjs/auth/blob/main/src/authenticator.ts)实例的引用。了解更多关于 [身份验证 ](../authentication/introduction.md) 的信息.

</dd>

<dt>

ctx.view

</dt>

<dd>

Edge 渲染器实例的引用。在 [视图和模板指南](../views-and-templates/introduction.md#using-edge) 中了解更多关于 Edge 的信息

</dd>

<dt>

ctx\.ally

</dt>

<dd>

[Ally Manager class](https://github.com/adonisjs/ally/blob/main/src/ally_manager.ts) 实例的引用，用于在应用中实现社交登录。了解更多关于 [Ally](../authentication/social_authentication.md)的信息

</dd>

<dt>

ctx.bouncer

</dt>

<dd>

[Bouncer class](https://github.com/adonisjs/bouncer/blob/main/src/bouncer.ts). 实例的引用。了解更多关于 [授权](../security/authorization.md)的信息.

</dd>

<dt>

ctx.i18n

</dt>

<dd>

[I18n class](https://github.com/adonisjs/i18n/blob/main/src/i18n.ts). 实例的引用。 in [Internationalization](../digging_deeper/i18n.md) 的更多信息。.

</dd>

</dl>

## 扩展 HTTP 上下文

您可以使用宏或 getter 向 HTTP 上下文类添加自定义属性。如果您对宏的概念不熟悉，请先阅读 [ 扩展 AdonisJS 指南](./extending_the_framework.md)

```ts
import { HttpContext } from '@adonisjs/core/http'

HttpContext.macro('aMethod', function (this: HttpContext) {
  return value
})

HttpContext.getter('aProperty', function (this: HttpContext) {
  return value
})
```

由于宏和 getter 是在运行时添加的，因此您必须使用模块增强来通知 TypeScript 它们的类型。

```ts
import { HttpContext } from '@adonisjs/core/http'

// insert-start
declare module '@adonisjs/core/http' {
  export interface HttpContext {
    aMethod: () => ValueType
    aProperty: ValueType
  }
}
// insert-end

HttpContext.macro('aMethod', function (this: HttpContext) {
  return value
})

HttpContext.getter('aProperty', function (this: HttpContext) {
  return value
})
```

## 在测试期间创建虚拟上下文

您可以使用 `testUtils` 服务在测试期间创建一个虚拟 HTTP 上下文。

上下文实例未附加到任何路由；因此 `ctx.route` 和 `ctx.params` 值将是未定义的。但是，如果测试代码需要，您可以手动分配这些属性。

```ts
import testUtils from '@adonisjs/core/services/test_utils'

const ctx = testUtils.createHttpContext()
```

默认情况下， `createHttpContext` 方法对 `req` 和 `res` 对象使用假值。但是，您可以为这些属性定义自定义值，如以下示例所示。

```ts
import { createServer } from 'node:http'
import testUtils from '@adonisjs/core/services/test_utils'

createServer((req, res) => {
  const ctx = testUtils.createHttpContext({
    // highlight-start
    req,
    res,
    // highlight-end
  })
})
```

### 使用 HttpContext 工厂

`testUtils` 服务仅在 AdonisJS 应用程序内部可用；因此，如果您正在构建一个包并且需要访问虚拟 HTTP 上下文，则可以使用 [HttpContextFactory](https://github.com/adonisjs/http-server/blob/main/factories/http_context.ts#L30) class.

```ts
import { HttpContextFactory } from '@adonisjs/core/factories/http'
const ctx = new HttpContextFactory().create()
```
