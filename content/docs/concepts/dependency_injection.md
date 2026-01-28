---
summary: Learn about dependency injection in AdonisJS and how to use the IoC container to resolve dependencies.
---

# 依赖注入

在每个AdonisJS应用程序的核心是一个IoC容器，它几乎无需配置就可以构造类并解析依赖项。

IoC容器主要服务于以下两个用途：

- 向第一方和第三方包公开API，以从容器中注册和解析绑定(更多内容请参见 [稍后的绑定](#container-bindings)).
- 自动解析并将依赖项注入到类构造函数或类方法中。

让我们从向一个类注入依赖项开始。

## 基本示例

自动依赖注入依赖于[TypeScript传统装饰器实现](https://www.typescriptlang.org/docs/handbook/decorators.html) and the [Reflection metadata ](https://www.npmjs.com/package/reflect-metadata) API.

在下面的示例中，我们创建一个 `EchoService` 类，并在 `HomeController` 类中注入其实例。您可以复制粘贴代码示例来跟随操作。

### Step 1. 创建Service类

首先在 `app/services` 文件夹中创建 `EchoService` class

```ts
// title: app/services/echo_service.ts
export default class EchoService {
  respond() {
    return 'hello'
  }
}
```

### Step 2. 在控制器中注入服务

在 `app/controllers` 文件夹中创建一个新的HTTP控制器。或者，您可以使用 `node ace make:controller home` 命令.

在控制器文件中导入 `EchoService` 并将其作为构造函数依赖项接受。

```ts
// title: app/controllers/home_controller.ts
import EchoService from '#services/echo_service'

export default class HomeController {
  constructor(protected echo: EchoService) {}

  handle() {
    return this.echo.respond()
  }
}
```

### Step 3. 使用inject装饰器

为了使自动依赖解析工作，我们必须在 `HomeController` 类上使用 `@inject` 装饰器。

```ts
import EchoService from '#services/echo_service'
// insert-start
import { inject } from '@adonisjs/core'
// insert-end

// insert-start
@inject()
// insert-end
export default class HomeController {
  constructor(protected echo: EchoService) {}

  handle() {
    return this.echo.respond()
  }
}
```

就是这样！现在您可以将 `HomeController` 类绑定到路由，它将自动接收 `EchoService` 类的实例。

### 结论

您可以将 `@inject` 装饰器视为一个间谍，观察类构造函数或方法依赖项并告知容器。

当AdonisJS路由器要求容器构造 `HomeController`, 时，容器已经知道控制器的依赖项。

## 构造依赖树

目前， `EchoService` 类没有任何依赖项，在这种情况下使用容器来创建其实例可能看起来有些小题大做。

让我们更新类构造函数，使其接受 `HttpContext` 类的实例

```ts
// title: app/services/echo_service.ts
// insert-start
import { inject } from '@adonisjs/core'
import { HttpContext } from '@adonisjs/core/http'
// insert-end

// insert-start
@inject()
// insert-end
export default class EchoService {
  // insert-start
  constructor(protected ctx: HttpContext) {}
  // insert-end

  respond() {
    return `Hello from ${this.ctx.request.url()}`
  }
}
```

同样，我们必须在 `EchoService` class 上放置我们的间谍 ( `@inject` 装饰器) 来检查它的依赖项。

瞧，这就是我们要做的全部。无需更改控制器中的任何一行代码，您就可以重新运行代码， `EchoService` class 将接收 `HttpContext` 类的实例。

:::note

使用容器的好处在于，您可以拥有深度嵌套的依赖关系，而容器可以为您解析整个依赖树。唯一的条件是使用 `@inject` 装饰器。

:::

## 使用方法注入

方法注入用于将依赖项注入到类方法中。要使方法注入生效，您必须在方法签名之前放置 `@inject` 装饰器。

让我们继续前面的示例，将 `EchoService` 依赖项从 `HomeController` 构造函数移动到 `handle` 方法。

:::note

在控制器中使用方法注入时，请记住第一个参数接收到固定值（即HTTP上下文），其余参数通过容器解析。

:::

```ts
// title: app/controllers/home_controller.ts
import EchoService from '#services/echo_service'
import { inject } from '@adonisjs/core'

// delete-start
@inject()
// delete-end
export default class HomeController {
  // delete-start
  constructor(private echo: EchoService) {}
  // delete-end

  // insert-start
  @inject()
  handle(ctx, echo: EchoService) {
    return echo.respond()
  }
  // insert-end
}
```

就是这样！这次，EchoService类实例将被注入到handle方法中。

## 何时使用依赖注入

建议在您的项目中利用依赖注入，因为DI在应用程序的不同部分之间创建了松耦合。因此，代码库更容易测试和重构。

但是，您必须小心，不要将依赖注入的概念推向极端，以至于您失去了它的优势。例如：

- 您不应将像 `lodash` 这样的辅助库作为类的依赖项进行注入。直接导入并使用它们即可。
- 对于那些不太可能被交换或替换的组件，您的代码库可能不需要松耦合。例如，您可能更喜欢导入 `logger` 服务，而不是将 `Logger` 类作为依赖项注入。

## 直接使用容器

AdonisJS应用程序中的大多数类，如 **控制器、中间件、事件监听器、验证器和邮件发送器** , 都是使用容器构建的。因此，您可以利用 `@inject` 装饰器进行自动依赖注入。

对于需要自己使用容器构建类实例的情况，您可以使用 `container.make` 方法。

`container.make` 方法接受一个类构造函数，并在解析所有依赖项后返回其实例。

```ts
import { inject } from '@adonisjs/core'
import app from '@adonisjs/core/services/app'

class EchoService {}

@inject()
class SomeService {
  constructor(public echo: EchoService) {}
}

/**
 * 与创建类的新实例相同，但
 * 将具有自动DI的优势
 */
const service = await app.container.make(SomeService)

console.log(service instanceof SomeService)
console.log(service.echo instanceof EchoService)
```

您可以使用 `container.call` 方法将依赖项注入到方法中。 The `container.call` 方法接受以下参数。

1. 类实例。
2. 要在类实例上运行的方法名称。容器将解析依赖项并将它们传递给该方法
3. 可选的固定参数数组，传递给该方法。

```ts
class EchoService {}

class SomeService {
  @inject()
  run(echo: EchoService) {}
}

const service = await app.container.make(SomeService)

/**
 * Echo类的实例将被传递
 * 给run方法
 */
await app.container.call(service, 'run')
```

## 容器绑定

容器绑定是IoC容器存在于AdonisJS中的主要原因之一。绑定充当您安装的包与应用程序之间的桥梁。

绑定本质上是键值对，键是绑定的唯一标识符，值是返回值的工厂函数。

- 绑定名称可以是 `string`, `symbol`, 或类构造函数。
- 工厂函数可以异步执行，且必须返回一个值。

您可以使用 `container.bind` 方法注册容器绑定。以下是向容器注册和解析绑定的简单示例。

```ts
import app from '@adonisjs/core/services/app'

class MyFakeCache {
  get(key: string) {
    return `${key}!`
  }
}

app.container.bind('cache', function () {
  return new MyFakeCache()
})

const cache = await app.container.make('cache')
console.log(cache.get('foo')) // 返回 foo!
```

### 何时使用容器绑定?

容器绑定用于特定用例，如为包导出的单例服务注册，或在自动依赖注入不足时自构建类实例。

我们建议您不要通过将所有内容注册到容器中来让应用程序变得不必要的复杂。相反，在使用容器绑定之前，请在应用程序代码中查找特定的用例。

以下是框架包内部使用容器绑定的一些示例。

- [在容器中注册BodyParserMiddleware](https://github.com/adonisjs/core/blob/main/providers/app_provider.ts#L134-L139): 由于中间件类需要存储在 `config/bodyparser.ts` 文件中的配置，因此无法使用自动依赖注入。在这种情况下，我们通过注册为绑定来手动构建中间件类实例。
- [将Encryption服务注册为单例](https://github.com/adonisjs/core/blob/main/providers/app_provider.ts#L97-L100): Encryption类需要存储在 `config/app.ts` 文件中的 `appKey` 因此我们使用容器绑定作为桥梁，从用户应用程序读取 `appKey` 并配置Encryption类的单例实例。

:::important

JavaScript生态系统中不常用容器绑定的概念。因此，随时 [加入我们的Discord社区](https://discord.gg/vDcEjq6) 来澄清您的疑问

:::

### 在工厂函数中解析绑定

您可以在绑定工厂函数内从容器中解析其他绑定。例如，如果 `MyFakeCache` 类需要来自 `config/cache.ts` 文件的配置，您可以按以下方式访问它。

```ts
this.app.container.bind('cache', async (resolver) => {
  const configService = await resolver.make('config')
  const cacheConfig = configService.get<any>('cache')

  return new MyFakeCache(cacheConfig)
})
```

### 单例

单例是工厂函数仅被调用一次的绑定，返回值将在应用程序生命周期内被缓存。

您可以使用 `container.singleton` 方法注册单例绑定。

```ts
this.app.container.singleton('cache', async (resolver) => {
  const configService = await resolver.make('config')
  const cacheConfig = configService.get<any>('cache')

  return new MyFakeCache(cacheConfig)
})
```

### 绑定值

您可以使用 `container.bindValue` 方法直接将值绑定到容器。

```ts
this.app.container.bindValue('cache', new MyFakeCache())
```

### 别名

您可以使用 `alias` 方法为绑定定义别名。该方法接受别名名称作为第一个参数，现有绑定或类构造函数的引用作为别名值。

```ts
this.app.container.singleton(MyFakeCache, async () => {
  return new MyFakeCache()
})

this.app.container.alias('cache', MyFakeCache)
```

### 为绑定定义静态类型

您可以使用 [TypeScript声明合并](https://www.typescriptlang.org/docs/handbook/declaration-merging.html) 为绑定定义静态类型信息。

类型在 `ContainerBindings` 接口上定义为键值对。

```ts
declare module '@adonisjs/core/types' {
  interface ContainerBindings {
    cache: MyFakeCache
  }
}
```

如果您创建包，可以在服务提供商文件中编写上述代码块。

在您的AdonisJS应用程序中，您可以在 `types/container.ts` 文件中编写上述代码块。

## 创建抽象层

容器允许您为应用程序创建抽象层。您可以为接口定义绑定，并将其解析为具体实现。

:::note
此方法在您想为应用程序应用六边形架构（也称为端口和适配器原则）时很有用
:::

由于TypeScript接口在运行时不存在，您必须为接口使用抽象类构造函数。

```ts
export abstract class PaymentService {
  abstract charge(amount: number): Promise<void>
  abstract refund(amount: number): Promise<void>
}
```

接下来，您可以创建 `PaymentService` 接口的具体实现。

```ts
import { PaymentService } from '#contracts/payment_service'

export class StripePaymentService implements PaymentService {
  async charge(amount: number) {
    // 使用Stripe收费
  }

  async refund(amount: number) {
    // 使用Stripe退款
  }
}
```

现在，您可以将 `PaymentService` 接口和 `StripePaymentService` 具体实现在容器中注册到您的 `AppProvider` 中.

```ts
// title: providers/app_provider.ts
import { PaymentService } from '#contracts/payment_service'

export default class AppProvider {
  async boot() {
    const { StripePaymentService } =
      await import('#services/stripe_payment_service')

    this.app.container.bind(PaymentService, () => {
      return this.app.container.make(StripePaymentService)
    })
  }
}
```

最后，您可以从容器中解析 `PaymentService` 接口并在应用程序中使用它。

```ts
import { PaymentService } from '#contracts/payment_service'

@inject()
export default class PaymentController {
  constructor(private paymentService: PaymentService) {}

  async charge() {
    await this.paymentService.charge(100)

    // ...
  }
}
```

## 测试期间切换实现

当您依靠容器来解析依赖树时，您对树中类的控制较少。因此，模拟/伪造这些类可能会变得更加困难。

在以下示例中， `UsersController.index` 方法接受 `UserService` 类的实例，我们使用 `@inject` 装饰器来解析依赖项并将其提供给 `index` 方法.

```ts
import UserService from '#services/user_service'
import { inject } from '@adonisjs/core'

export default class UsersController {
  @inject()
  index(service: UserService) {}
}
```

假设在测试期间，您不想使用实际的 `UserService` 因为它会发出外部HTTP请求。相反，您想要使用假实现。

但首先，看看您可能编写的用于测试 `UsersController`的代码.

```ts
import UserService from '#services/user_service'

test('get all users', async ({ client }) => {
  const response = await client.get('/users')

  response.assertBody({
    data: [{ id: 1, username: 'virk' }],
  })
})
```

在上面的测试中，我们通过HTTP请求与 `UsersController` 交互，并没有直接控制它。

容器提供了简单的API来用假实现替换类。您可以使用 `container.swap` 方法定义交换。

`container.swap` 方法接受您想要交换的类构造函数，然后是一个返回替代实现的工厂函数。

```ts
import UserService from '#services/user_service'
// insert-start
import app from '@adonisjs/core/services/app'
// insert-end

test('get all users', async ({ client }) => {
  // insert-start
  class FakeService extends UserService {
    all() {
      return [{ id: 1, username: 'virk' }]
    }
  }

  app.container.swap(UserService, () => {
    return new FakeService()
  })
  // insert-end

  const response = await client.get('users')
  response.assertBody({
    data: [{ id: 1, username: 'virk' }],
  })
})
```

一旦定义了交换，容器将使用它而不是实际类。您可以使用 `container.restore` 方法恢复原始实现。

```ts
app.container.restore(UserService)

// 恢复UserService和PostService
app.container.restoreAll([UserService, PostService])

// 恢复所有
app.container.restoreAll()
```

## 上下文依赖

上下文依赖允许您定义如何为给定类解析依赖项。例如，您有两个服务依赖于Drive Disk类。

```ts
import { Disk } from '@adonisjs/drive'

export default class UserService {
  constructor(protected disk: Disk) {}
}
```

```ts
import { Disk } from '@adonisjs/drive'

export default class PostService {
  constructor(protected disk: Disk) {}
}
```

您希望 `UserService` 接收使用GCS驱动程序的磁盘实例，而 `PostService` 接收使用S3驱动程序的磁盘实例。您可以使用上下文依赖实现这一点。

以下代码必须写在服务提供商 `register` 方法中。

```ts
import { Disk } from '@adonisjs/drive'
import UserService from '#services/user_service'
import PostService from '#services/post_service'
import { ApplicationService } from '@adonisjs/core/types'

export default class AppProvider {
  constructor(protected app: ApplicationService) {}

  register() {
    this.app.container
      .when(UserService)
      .asksFor(Disk)
      .provide(async (resolver) => {
        const driveManager = await resolver.make('drive')
        return drive.use('gcs')
      })

    this.app.container
      .when(PostService)
      .asksFor(Disk)
      .provide(async (resolver) => {
        const driveManager = await resolver.make('drive')
        return drive.use('s3')
      })
  }
}
```

## 容器钩子

您可以使用容器的 `resolving` 钩子修改/扩展 `container.make` 方法的返回值。

通常，您会在服务提供商内部使用钩子，尝试扩展特定绑定时。例如，数据库提供商使用 `resolving` 钩子来注册额外的数据库驱动验证规则。

```ts
import { ApplicationService } from '@adonisjs/core/types'

export default class DatabaseProvider {
  constructor(protected app: ApplicationService) {}

  async boot() {
    this.app.container.resolving('validator', (validator) => {
      validator.rule('unique', implementation)
      validator.rule('exists', implementation)
    })
  }
}
```

## 容器事件

容器在解析绑定或构造类实例后发出 `container_binding:resolved` 事件. `event.binding` 属性将是字符串（绑定名称）或类构造函数， `event.value` 属性是已解析的值.

```ts
import emitter from '@adonisjs/core/services/emitter'

emitter.on('container_binding:resolved', (event) => {
  console.log(event.binding)
  console.log(event.value)
})
```

## 参见

- [容器README文件](https://github.com/adonisjs/fold/blob/develop/README.md) 以框架无关的方式介绍了容器API。
- [为什么需要IoC容器？](https://github.com/thetutlage/meta/discussions/4) 在这篇文章中，框架创建者分享了他使用IoC容器的理由。
