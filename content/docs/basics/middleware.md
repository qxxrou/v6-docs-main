---
summary: Learn about middleware in AdonisJS, how to create them, and how to assign them to routes and route groups.
---

# 中间件

中间件是在HTTP请求到达路由处理器之前执行的一系列函数。链中的每个函数都可以结束请求或转发到下一个中间件。

典型的AdonisJS应用程序使用中间件来**解析请求体**、**管理用户会话**、**认证请求**、**提供静态资源**等。

你也可以创建自定义中间件来在HTTP请求期间执行其他任务。

## 中间件栈

为了更好地控制中间件管道的执行，AdonisJS将中间件栈分为以下三组。

### 服务器中间件栈

服务器中间件在每个HTTP请求上运行，即使你没有为当前请求的URL定义任何路由。

它们非常适合为应用程序添加不依赖于框架路由系统的额外功能。例如，静态资源中间件被注册为服务器中间件。

你可以在`start/kernel.ts`文件中使用`server.use`方法注册服务器中间件。

```ts
import server from '@adonisjs/core/services/server'

server.use([() => import('@adonisjs/static/static_middleware')])
```

---

### 路由中间件栈

路由中间件也称为全局中间件。它们在每个有匹配路由的HTTP请求上执行。

Bodyparser、auth和session中间件被注册在路由中间件栈下。

你可以在`start/kernel.ts`文件中使用`router.use`方法注册路由中间件。

```ts
import router from '@adonisjs/core/services/router'

router.use([() => import('@adonisjs/core/bodyparser_middleware')])
```

---

### 命名中间件集合

命名中间件是一组不会执行的中间件，除非显式分配给路由或组。

我们建议你创建专用的中间件类，将它们存储在命名中间件集合中，然后将它们分配给路由，而不是在路由文件中定义内联回调中间件。

你可以在`start/kernel.ts`文件中使用`router.named`方法定义命名中间件。确保导出命名集合以便能够[在路由文件中使用](#为路由和路由组分配中间件)。

```ts
import router from '@adonisjs/core/services/router'

router.named({
  auth: () => import('#middleware/auth_middleware'),
})
```

## 创建中间件

中间件存储在`./app/middleware`目录中，你可以通过运行`make:middleware` ace命令创建一个新的中间件文件。

另请参阅：[Make middleware command](../references/commands.md#makemiddleware)

```sh
node ace make:middleware user_location
```

上面的命令将在中间件目录下创建`user_location_middleware.ts`文件。

中间件表示为带有`handle`方法的类。在执行期间，AdonisJS将自动调用此方法，并将[HttpContext](../concepts/http_context.md)作为第一个参数传递给它。

```ts
// title: app/middleware/user_location_middleware.ts
import { HttpContext } from '@adonisjs/core/http'
import { NextFn } from '@adonisjs/core/types/http'

export default class UserLocationMiddleware {
  async handle(ctx: HttpContext, next: NextFn) {}
}
```

在`handle`方法中，中间件必须决定是继续请求、通过发送响应结束请求还是引发异常中止请求。

### 中止请求

如果中间件引发异常，所有后续中间件和路由处理器都不会执行，异常将被传递给全局异常处理器。

```ts
import { Exception } from '@adonisjs/core/exceptions'
import { NextFn } from '@adonisjs/core/types/http'

export default class UserLocationMiddleware {
  async handle(ctx: HttpContext, next: NextFn) {
    throw new Exception('Aborting request')
  }
}
```

### 继续请求

你必须调用`next`方法才能继续请求。否则，中间件栈内的其余操作将不会执行。

```ts
export default class UserLocationMiddleware {
  async handle(ctx: HttpContext, next: NextFn) {
    // Call the `next` function to continue
    await next()
  }
}
```

### 发送响应，不调用`next`方法

最后，你可以通过发送响应来结束请求。在这种情况下，不要调用`next`方法。

```ts
export default class UserLocationMiddleware {
  async handle(ctx: HttpContext, next: NextFn) {
    // send response + do not call next
    ctx.response.send('Ending request')
  }
}
```

## 为路由和路由组分配中间件

默认情况下，命名中间件集合是未使用的，你必须显式将它们分配给路由或路由组。

在下面的示例中，我们首先导入`middleware`集合，并将`userLocation`中间件分配给路由。

```ts
import router from '@adonisjs/core/services/router'
import { middleware } from '#start/kernel'

router.get('posts', () => {}).use(middleware.userLocation())
```

可以将多个中间件应用为数组，或者多次调用`use`方法。

```ts
router
  .get('posts', () => {})
  .use([middleware.userLocation(), middleware.auth()])
```

同样，你也可以将中间件分配给路由组。组中间件将自动应用于所有组路由。

```ts
import router from '@adonisjs/core/services/router'
import { middleware } from '#start/kernel'

router
  .group(() => {
    router.get('posts', () => {})
    router.get('users', () => {})
    router.get('payments', () => {})
  })
  .use(middleware.userLocation())
```

## 中间件参数

注册在命名中间件集合下的中间件可以接受额外的参数作为`handle`方法参数的一部分。例如，`auth`中间件接受认证守卫作为配置选项。

```ts
type AuthGuards = 'web' | 'api'

export default class AuthMiddleware {
  async handle(ctx, next, options: { guard: AuthGuards }) {}
}
```

在将中间件分配给路由时，你可以指定要使用的守卫。

```ts
import router from '@adonisjs/core/services/router'
import { middleware } from '#start/kernel'

router.get('payments', () => {}).use(middleware.auth({ guard: 'web' }))
```

## 依赖注入

中间件类使用[IoC容器](../concepts/dependency_injection.md)实例化；因此，你可以在中间件构造函数中键入提示依赖项，容器将为你注入它们。

假设你有一个`GeoIpService`类来从请求IP查找用户位置，你可以使用`@inject`装饰器将其注入到中间件中。

```ts
// title: app/services/geoip_service.ts
export class GeoIpService {
  async lookup(ipAddress: string) {
    // lookup location and return
  }
}
```

```ts
import { inject } from '@adonisjs/core'
import { GeoIpService } from '#services/geoip_service'
import type { HttpContext } from '@adonisjs/core/http'
import type { NextFn } from '@adonisjs/core/types/http'

@inject()
export default class UserLocationMiddleware {
  constructor(protected geoIpService: GeoIpService) {}

  async handle(ctx: HttpContext, next: NextFn) {
    const ip = ctx.request.ip()
    ctx.location = await this.geoIpService.lookup(ip)
  }
}
```

## 中间件执行流程

AdonisJS的中间件层构建在[责任链](https://refactoring.guru/design-patterns/chain-of-responsibility)设计模式之上。中间件有两个执行阶段：**下游阶段**和**上游阶段**。

- 下游阶段是在调用`next`方法之前编写的代码块。在这个阶段，你处理请求。
- 上游阶段是在调用`next`方法之后可能编写的代码块。在这个阶段，你可以检查响应或完全更改它。

![](./middleware_flow.jpeg)

## 中间件和异常处理

AdonisJS自动捕获中间件管道或路由处理器引发的异常，并使用[全局异常处理器](./exception_handling.md)将其转换为HTTP响应。

因此，你不必将`next`函数调用包装在`try/catch`语句中。此外，自动异常处理确保中间件的上游逻辑总是在`next`函数调用后执行。

## 从中间件修改响应

中间件的上游阶段可以修改响应体、头和状态码。这样做将丢弃路由处理器或任何其他中间件设置的旧响应。

在修改响应之前，你必须确保你正在处理正确的响应类型。以下是`Response`类中的响应类型列表。

- **标准响应**是指使用`response.send`方法发送数据值。其值可能是`Array`、`Object`、`String`、`Boolean`或`Buffer`。
- **流响应**是指使用`response.stream`方法将流管道传输到响应套接字。
- **文件下载响应**是指使用`response.download`方法下载文件。

根据响应类型，你将有权访问或无权访问特定的响应属性。

### 处理标准响应

修改标准响应时，你可以使用`response.content`属性访问它。确保首先检查`content`是否存在。

```ts
import { HttpContext } from '@adonisjs/core/http'
import { NextFn } from '@adonisjs/core/types/http'

export default class {
  async handle({ response }: HttpContext, next: NextFn) {
    await next()

    if (response.hasContent) {
      console.log(response.content)
      console.log(typeof response.content)

      response.send(newResponse)
    }
  }
}
```

### 处理流响应

使用`response.stream`方法设置的响应流不会立即管道传输到传出的[HTTP响应](https://nodejs.org/api/http.html#class-httpserverresponse)。相反，AdonisJS会等待路由处理器和中间件管道完成。

因此，在中间件内部，你可以用新流替换现有流，或定义事件处理程序来监控流。

```ts
import { HttpContext } from '@adonisjs/core/http'
import { NextFn } from '@adonisjs/core/types/http'

export default class {
  async handle({ response }: HttpContext, next: NextFn) {
    await next()

    if (response.hasStream) {
      response.outgoingStream.on('data', (chunk) => {
        console.log(chunk)
      })
    }
  }
}
```

### 处理文件下载

使用`response.download`和`response.attachment`方法执行的文件下载会将下载过程推迟到路由处理器和中间件管道完成。

因此，在中间件内部，你可以替换要下载的文件的路径。

```ts
import { HttpContext } from '@adonisjs/core/http'
import { NextFn } from '@adonisjs/core/types/http'

export default class {
  async handle({ response }: HttpContext, next: NextFn) {
    await next()

    if (response.hasFileToStream) {
      console.log(response.fileToStream.generateEtag)
      console.log(response.fileToStream.path)
    }
  }
}
```

## 测试中间件类

将中间件创建为类允许你轻松地隔离测试中间件（也称为单元测试中间件）。有几种不同的方法来测试中间件。让我们探讨所有可用的选项。

最简单的选择是创建中间件类的新实例，并使用HTTP上下文和`next`回调函数调用`handle`方法。

```ts
import testUtils from '@adonisjs/core/services/test_utils'
import GeoIpService from '#services/geoip_service'
import UserLocationMiddleware from '#middleware/user_location_middleware'

const middleware = new UserLocationMiddleware(new GeoIpService())

const ctx = testUtils.createHttpContext()
await middleware.handle(ctx, () => {
  console.log('Next function invoked')
})
```

`testUtils`服务仅在AdonisJS应用程序启动后可用。但是，如果你正在包中测试中间件，你可以使用`HttpContextFactory`类创建一个虚拟HTTP上下文实例，而无需启动应用程序。

另请参阅：[CORS中间件测试](https://github.com/adonisjs/cors/blob/main/tests/cors_middleware.spec.ts#L24-L41)的真实示例。

```ts
import {
  RequestFactory,
  ResponseFactory,
  HttpContextFactory,
} from '@adonisjs/core/factories/http'

const request = new RequestFactory().create()
const response = new ResponseFactory().create()
const ctx = new HttpContextFactory().merge({ request, response }).create()

await middleware.handle(ctx, () => {
  console.log('Next function invoked')
})
```

### 使用服务器管道

如果你的中间件依赖于其他中间件先执行，你可以使用`server.pipeline`方法组合中间件管道。

- `server.pipeline`方法接受中间件类的数组。
- 类实例使用IoC容器创建。
- 执行流程与HTTP请求期间中间件的原始执行流程相同。

```ts
import testUtils from '@adonisjs/core/services/test_utils'
import server from '@adonisjs/core/services/server'
import UserLocationMiddleware from '#middleware/user_location_middleware'

const pipeline = server.pipeline([UserLocationMiddleware])

const ctx = testUtils.createHttpContext()
await pipeline.run(ctx)
```

你可以在调用`pipeline.run`方法之前定义`finalHandler`和`errorHandler`函数。

- 最终处理器在所有中间件执行后执行。当任何中间件在不调用`next`方法的情况下结束链时，最终处理器不会执行。
- 如果中间件引发异常，则执行错误处理器。上游流程将在错误处理器被调用后开始。

```ts
const ctx = testUtils.createHttpContext()

await pipeline
  .finalHandler(() => {
    console.log('all middleware called next')
    console.log('the upstream logic starts from here')
  })
  .errorHandler((error) => {
    console.log('an exception was raised')
    console.log('the upstream logic starts from here')
  })
  .run(ctx)

console.log('pipeline executed')
```

`server`服务在应用程序启动后可用。但是，如果你正在创建包，你可以使用`ServerFactory`创建Server类的实例，而无需启动应用程序。

```ts
import { ServerFactory } from '@adonisjs/core/factories/http'

const server = new ServerFactory().create()
const pipeline = server.pipeline([UserLocationMiddleware])
```
