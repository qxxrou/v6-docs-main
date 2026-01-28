---
summary: 异常是在HTTP请求生命周期中引发的错误。AdonisJS提供了强大的异常处理机制，可以将异常转换为HTTP响应并将其报告给日志记录器。
---

# 异常处理

在HTTP请求期间引发的异常由定义在 `./app/exceptions/handler.ts` 文件中的 `HttpExceptionHandler` 处理。在此文件中，您可以决定如何将异常转换为响应，并使用日志记录器记录它们或将它们报告给外部日志记录提供程序。

`HttpExceptionHandler` 扩展了 [ExceptionHandler](https://github.com/adonisjs/http-server/blob/main/src/exception_handler.ts) 类，该类完成了处理错误的所有繁重工作，并为您提供了高级API来调整报告和渲染行为。

```ts
import app from '@adonisjs/core/services/app'
import { HttpContext, ExceptionHandler } from '@adonisjs/core/http'

export default class HttpExceptionHandler extends ExceptionHandler {
  protected debug = !app.inProduction
  protected renderStatusPages = app.inProduction

  async handle(error: unknown, ctx: HttpContext) {
    return super.handle(error, ctx)
  }

  async report(error: unknown, ctx: HttpContext) {
    return super.report(error, ctx)
  }
}
```

## 将错误处理程序分配给服务器

错误处理程序在 `start/kernel.ts` 文件中注册到AdonisJS HTTP服务器。我们使用在 `package.json` 文件中定义的 `#exceptions` 别名延迟导入HTTP处理程序。

```ts
server.errorHandler(() => import('#exceptions/handler'))
```

## 处理异常

异常由异常处理程序类上的 `handle` 方法处理。默认情况下，在处理错误时执行以下步骤。

- 检查错误实例是否具有 `handle` 方法。如果有，调用 [error.handle](#defining-the-handle-method) 方法并返回其响应。
- 检查是否为 `error.status` 代码定义了状态页面。如果有，渲染状态页面。
- 否则，使用内容协商渲染器渲染异常。

如果您想以不同方式处理特定异常，可以在 `handle` 方法中执行此操作。确保使用 `ctx.response.send` 方法发送响应，因为 `handle` 方法的返回值将被丢弃。

```ts
import { errors } from '@vinejs/vine'

export default class HttpExceptionHandler extends ExceptionHandler {
  async handle(error: unknown, ctx: HttpContext) {
    if (error instanceof errors.E_VALIDATION_ERROR) {
      ctx.response.status(422).send(error.messages)
      return
    }

    return super.handle(error, ctx)
  }
}
```

### 状态页面

状态页面是您要为给定或一系列状态代码渲染的模板集合。

状态代码范围可以定义为字符串表达式。两个点分隔起始和结束状态代码（`..`）。

如果您正在创建JSON服务器，可能不需要状态页面。

```ts
import {
  StatusPageRange,
  StatusPageRenderer,
} from '@adonisjs/http-server/types'

export default class HttpExceptionHandler extends ExceptionHandler {
  protected statusPages: Record<StatusPageRange, StatusPageRenderer> = {
    '404': (_, { view }) => view.render('errors/not-found'),
    '500..599': (_, { view }) => view.render('errors/server-error'),
  }
}
```

### 调试模式

内容协商渲染器处理未自行处理且未转换为状态页面的异常。

内容协商渲染器支持调试模式。它们可以使用 [Youch](https://www.npmjs.com/package/youch) npm包在调试模式下解析和漂亮地打印错误。

您可以使用异常处理程序类上的 `debug` 属性切换调试模式。但是，建议在生产环境中关闭调试模式，因为它会暴露有关您的应用的敏感信息。

```ts
export default class HttpExceptionHandler extends ExceptionHandler {
  protected debug = !app.inProduction
}
```

## 报告异常

异常处理程序类上的 `report` 方法处理异常的报告。

该方法接收错误作为第一个参数，[HTTP上下文](../concepts/http_context.md)作为第二个参数。您不应该从 `report` 方法写入响应，而应仅使用上下文读取请求信息。

### 记录异常

默认情况下，所有异常都使用 [日志记录器](../digging_deeper/logger.md) 进行报告。

- 状态代码在 `400..499` 范围内的异常以 `warning` 级别记录。
- 状态代码 `>=500` 的异常以 `error` 级别记录。
- 所有其他异常以 `info` 级别记录。

您可以通过从 `context` 方法返回对象来向日志消息添加自定义属性。

```ts
export default class HttpExceptionHandler extends ExceptionHandler {
  protected context(ctx: HttpContext) {
    return {
      requestId: ctx.requestId,
      userId: ctx.auth.user?.id,
      ip: ctx.request.ip(),
    }
  }
}
```

### 忽略状态代码

您可以通过 `ignoreStatuses` 属性定义状态代码数组来忽略异常不被报告。

```ts
export default class HttpExceptionHandler extends ExceptionHandler {
  protected ignoreStatuses = [401, 400, 422, 403]
}
```

### 忽略错误

您还可以通过定义错误代码数组或要忽略的错误类来忽略异常。

```ts
import { errors } from '@adonisjs/core'
import { errors as sessionErrors } from '@adonisjs/session'

export default class HttpExceptionHandler extends ExceptionHandler {
  protected ignoreCodes = ['E_ROUTE_NOT_FOUND', 'E_INVALID_SESSION']
}
```

可以使用 `ignoreExceptions` 属性忽略异常类数组。

```ts
import { errors } from '@adonisjs/core'
import { errors as sessionErrors } from '@adonisjs/session'

export default class HttpExceptionHandler extends ExceptionHandler {
  protected ignoreExceptions = [
    errors.E_ROUTE_NOT_FOUND,
    sessionErrors.E_INVALID_SESSION,
  ]
}
```

### 自定义shouldReport方法

忽略状态代码或异常的逻辑写在 [`shouldReport` 方法](https://github.com/adonisjs/http-server/blob/main/src/exception_handler.ts#L155) 内。如果需要，您可以覆盖此方法并定义自定义逻辑来忽略异常。

```ts
import { HttpError } from '@adonisjs/core/types/http'

export default class HttpExceptionHandler extends ExceptionHandler {
  protected shouldReport(error: HttpError) {
    // return a boolean
  }
}
```

## 自定义异常

您可以使用 `make:exception` ace命令创建异常类。异常扩展了 `@adonisjs/core` 包中的 `Exception` 类。

另请参阅：[Make exception命令](../references/commands.md#makeexception)

```sh
node ace make:exception UnAuthorized
```

```ts
import { Exception } from '@adonisjs/core/exceptions'

export default class UnAuthorizedException extends Exception {}
```

您可以通过创建异常的新实例来引发异常。引发异常时，您可以为异常分配自定义 **错误代码** 和 **状态代码**。

```ts
import UnAuthorizedException from '#exceptions/unauthorized_exception'

throw new UnAuthorizedException('You are not authorized', {
  status: 403,
  code: 'E_UNAUTHORIZED',
})
```

错误代码和状态代码也可以定义为异常类上的静态属性。如果在引发异常时未定义自定义值，则将使用静态值。

```ts
import { Exception } from '@adonisjs/core/exceptions'
export default class UnAuthorizedException extends Exception {
  static status = 403
  static code = 'E_UNAUTHORIZED'
}
```

### 定义 `handle` 方法

要自行处理异常，您可以在异常类上定义 `handle` 方法。此方法应使用 `ctx.response.send` 方法将错误转换为HTTP响应。

`error.handle` 方法接收错误实例作为第一个参数，HTTP上下文作为第二个参数。

```ts
import { Exception } from '@adonisjs/core/exceptions'
import { HttpContext } from '@adonisjs/core/http'

export default class UnAuthorizedException extends Exception {
  async handle(error: this, ctx: HttpContext) {
    ctx.response.status(error.status).send(error.message)
  }
}
```

### 定义 `report` 方法

您可以在异常类上实现 `report` 方法来自行处理异常报告。report方法接收错误实例作为第一个参数，HTTP上下文作为第二个参数。

```ts
import { Exception } from '@adonisjs/core/exceptions'
import { HttpContext } from '@adonisjs/core/http'

export default class UnAuthorizedException extends Exception {
  async report(error: this, ctx: HttpContext) {
    ctx.logger.error({ err: error }, error.message)
  }
}
```

## 缩小错误类型

框架核心和其他官方包导出它们引发的异常。您可以使用 `instanceof` 检查来验证错误是否是特定异常的实例。例如：

```ts
import { errors } from '@adonisjs/core'

try {
  router.builder().make('articles.index')
} catch (error: unknown) {
  if (error instanceof errors.E_CANNOT_LOOKUP_ROUTE) {
    // handle error
  }
}
```

## 已知错误

请查看 [异常参考指南](../references/exceptions.md) 以查看已知错误列表。
