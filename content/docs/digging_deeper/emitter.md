---
summary: 基于 emittery 创建的内置事件发射器。异步调度事件并修复了 Node.js 默认事件发射器的许多常见问题。
---

# 事件发射器

AdonisJS 有一个基于 [emittery](https://github.com/sindresorhus/emittery) 创建的内置事件发射器。Emittery 异步调度事件并 [修复了 Node.js 默认事件发射器的许多常见问题](https://github.com/sindresorhus/emittery#how-is-this-different-than-the-built-in-eventemitter-in-nodejs)。

AdonisJS 通过额外的功能进一步增强了 emittery。

- 通过定义事件列表及其关联的数据类型来提供静态类型安全。
- 支持基于类的事件和监听器。将监听器移动到专用文件中可以使您的代码库保持整洁且易于测试。
- 能够在测试期间模拟事件。

## 基本用法

事件监听器在 `start/events.ts` 文件中定义。您可以使用 `make:preload` ace 命令创建此文件。

```sh
node ace make:preload events
```

您必须使用 `emitter.on` 来监听事件。该方法接受事件名称作为第一个参数，监听器作为第二个参数。

```ts
// title: start/events.ts
import emitter from '@adonisjs/core/services/emitter'

emitter.on('user:registered', function (user) {
  console.log(user)
})
```

一旦定义了事件监听器，您就可以使用 `emitter.emit` 方法发出 `user:registered` 事件。它将事件名称作为第一个参数，事件数据作为第二个参数。

```ts
import emitter from '@adonisjs/core/services/emitter'

export default class UsersController {
  async store() {
    const user = await User.create(data)
    emitter.emit('user:registered', user)
  }
}
```

您可以使用 `emitter.once` 只监听一次事件。

```ts
emitter.once('user:registered', function (user) {
  console.log(user)
})
```

## 使事件类型安全

AdonisJS 强制要求为您要在应用程序内发出的每个事件定义静态类型。这些类型在 `types/events.ts` 文件中注册。

在以下示例中，我们将 `User` 模型注册为 `user:registered` 事件的数据类型。

:::note

如果您发现为每个事件定义类型很麻烦，您可以切换到 [基于类的事件](#class-based-events)。

:::

```ts
import User from '#models/User'

declare module '@adonisjs/core/types' {
  interface EventsList {
    'user:registered': User
  }
}
```

## 基于类的监听器

与 HTTP 控制器一样，监听器类提供了一个抽象层，可以将内联事件监听器移动到专用文件中。监听器类存储在 `app/listeners` 目录中，您可以使用 `make:listener` 命令创建新的监听器。

另请参阅：[Make listener 命令](../references/commands.md#makelistener)

```sh
node ace make:listener sendVerificationEmail
```

监听器文件使用 `class` 声明和 `handle` 方法进行脚手架。在这个类中，您可以定义额外的方法来监听多个事件（如果需要）。

```ts
import User from '#models/user'

export default class SendVerificationEmail {
  handle(user: User) {
    // 发送邮件
  }
}
```

作为最后一步，您必须在 `start/events.ts` 文件中将监听器类绑定到事件。您可以使用 `#listeners` 别名导入监听器。别名是使用 Node.js 的 [子路径导入功能](../getting_started/folder_structure.md#the-sub-path-imports) 定义的。

```ts
// title: start/events.ts
import emitter from '@adonisjs/core/services/emitter'
import SendVerificationEmail from '#listeners/send_verification_email'

emitter.on('user:registered', [SendVerificationEmail, 'handle'])
```

### 延迟加载监听器

建议延迟加载监听器以加快应用程序启动阶段。延迟加载就像将导入语句移动到函数后面并使用动态导入一样简单。

```ts
import emitter from '@adonisjs/core/services/emitter'
// delete-start
import SendVerificationEmail from '#listeners/send_verification_email'
// delete-end
// insert-start
const SendVerificationEmail = () => import('#listeners/send_verification_email')
// insert-end

emitter.on('user:registered', [SendVerificationEmail, 'handle'])
```

### 依赖注入

:::warning

您不能在监听器类中注入 `HttpContext`。因为事件是异步处理的，监听器可能会在 HTTP 请求完成后运行。

:::

监听器类使用 [IoC 容器](../concepts/dependency_injection.md) 实例化；因此，您可以在类构造函数或处理事件的方法中键入提示依赖项。

在以下示例中，我们将 `TokensService` 作为构造函数参数进行类型提示。调用此监听器时，IoC 容器将注入 `TokensService` 类的实例。

```ts
// title: 构造函数注入
import { inject } from '@adonisjs/core'
import TokensService from '#services/tokens_service'

@inject()
export default class SendVerificationEmail {
  constructor(protected tokensService: TokensService) {}

  handle(user: User) {
    const token = this.tokensService.generate(user.email)
  }
}
```

在以下示例中，我们将 `TokensService` 注入到 handle 方法中。但是，请记住，第一个参数始终是事件有效负载。

```ts
// title: 方法注入
import { inject } from '@adonisjs/core'
import TokensService from '#services/tokens_service'
import UserRegistered from '#events/user_registered'

export default class SendVerificationEmail {
  @inject()
  handle(event: UserRegistered, tokensService: TokensService) {
    const token = tokensService.generate(event.user.email)
  }
}
```

## 基于类的事件

事件是事件标识符（通常是基于字符串的事件名称）和相关数据类型的组合。例如：

- `user:registered` 是事件标识符（又称事件名称）。
- User 模型的实例是事件数据。

基于类的事件将事件标识符和事件数据封装在同一个类中。类构造函数用作标识符，类的实例保存事件数据。

您可以使用 `make:event` 命令创建事件类。

另请参阅：[Make event 命令](../references/commands.md#makeevent)

```sh
node ace make:event UserRegistered
```

事件类没有行为。相反，它是事件的数据容器。您可以通过类构造函数接受事件数据，并将其作为实例属性提供。

```ts
// title: app/events/user_registered.ts
import { BaseEvent } from '@adonisjs/core/events'
import User from '#models/user'

export default class UserRegistered extends BaseEvent {
  constructor(public user: User) {
    super()
  }
}
```

### 监听基于类的事件

您可以使用 `emitter.on` 方法附加基于类的事件的监听器。第一个参数是事件类引用，后跟监听器。

```ts
import emitter from '@adonisjs/core/services/emitter'
import UserRegistered from '#events/user_registered'

emitter.on(UserRegistered, function (event) {
  console.log(event.user)
})
```

在以下示例中，我们同时使用基于类的事件和监听器。

```ts
import emitter from '@adonisjs/core/services/emitter'

import UserRegistered from '#events/user_registered'
const SendVerificationEmail = () => import('#listeners/send_verification_email')

emitter.on(UserRegistered, [SendVerificationEmail])
```

### 发出基于类的事件

您可以使用 `static dispatch` 方法发出基于类的事件。`dispatch` 方法接受事件类构造函数接受的参数。

```ts
import User from '#models/user'
import UserRegistered from '#events/user_registered'

export default class UsersController {
  async store() {
    const user = await User.create(data)

    /**
     * 分发/发出事件
     */
    UserRegistered.dispatch(user)
  }
}
```

## 简化事件监听体验

如果您决定使用基于类的事件和监听器，您可以使用 `emitter.listen` 方法来简化绑定监听器的过程。

`emitter.listen` 方法接受事件类作为第一个参数，基于类的监听器数组作为第二个参数。此外，注册的监听器必须有一个 `handle` 方法来处理事件。

```ts
import emitter from '@adonisjs/core/services/emitter'
import UserRegistered from '#events/user_registered'

emitter.listen(UserRegistered, [
  () => import('#listeners/send_verification_email'),
  () => import('#listeners/register_with_payment_provider'),
  () => import('#listeners/provision_account'),
])
```

## 处理错误

默认情况下，监听器引发的异常将导致 [unhandledRejection](https://nodejs.org/api/process.html#event-unhandledrejection)。因此，建议使用 `emitter.onError` 方法自行捕获和处理错误。

`emitter.onError` 方法接受一个回调，该回调接收事件名称、错误和事件数据。

```ts
import emitter from '@adonisjs/core/services/emitter'

emitter.onError((event, error, eventData) => {})
```

## 监听所有事件

您可以使用 `emitter.onAny` 方法监听所有事件。该方法接受监听器回调作为唯一参数。

```ts
import emitter from '@adonisjs/core/services/emitter'

emitter.onAny((name, event) => {
  console.log(name)
  console.log(event)
})
```

## 取消订阅事件

`emitter.on` 方法返回一个取消订阅函数，您可以调用该函数来删除事件监听器订阅。

```ts
import emitter from '@adonisjs/core/services/emitter'

const unsubscribe = emitter.on('user:registered', () => {})

// 删除监听器
unsubscribe()
```

您也可以使用 `emitter.off` 方法删除事件监听器订阅。该方法接受事件名称/类作为第一个参数，监听器的引用作为第二个参数。

```ts
import emitter from '@adonisjs/core/services/emitter'

function sendEmail() {}

// 监听事件
emitter.on('user:registered', sendEmail)

// 删除监听器
emitter.off('user:registered', sendEmail)
```

### emitter.offAny

`emitter.offAny` 移除监听所有事件的通配符监听器。

```ts
emitter.offAny(callback)
```

### emitter.clearListeners

`emitter.clearListeners` 方法移除给定事件的所有监听器。

```ts
// 基于字符串的事件
emitter.clearListeners('user:registered')

// 基于类的事件
emitter.clearListeners(UserRegistered)
```

### emitter.clearAllListeners

`emitter.clearAllListeners` 方法移除所有事件的所有监听器。

```ts
emitter.clearAllListeners()
```

## 可用事件列表

请查看 [事件参考指南](../references/events.md) 以查看可用事件的列表。

## 在测试期间模拟事件

事件监听器通常用于在给定操作后执行副作用。例如：向新注册的用户发送电子邮件，或在订单状态更新后发送通知。

编写测试时，您可能希望避免向测试期间创建的用户发送电子邮件。

您可以通过模拟事件发射器来禁用事件监听器。在以下示例中，我们在发出 HTTP 请求注册用户之前调用 `emitter.fake`。请求后，我们使用 `events.assertEmitted` 方法确保 `SignupController` 发出特定事件。

```ts
import emitter from '@adonisjs/core/services/emitter'
import UserRegistered from '#events/user_registered'

test.group('User signup', () => {
  test('create a user account', async ({ client, cleanup }) => {
    // highlight-start
    const events = emitter.fake()
    cleanup(() => {
      emitter.restore()
    })
    // highlight-end

    await client.post('signup').form({
      email: 'foo@bar.com',
      password: 'secret',
    })
  })

  // highlight-start
  // 断言事件已发出
  events.assertEmitted(UserRegistered)
  // highlight-end
})
```

- `event.fake` 方法返回 [EventBuffer](https://github.com/adonisjs/events/blob/main/src/events_buffer.ts) 类的实例，您可以使用它进行断言和查找发出的事件。
- `emitter.restore` 方法恢复 fake。恢复 fake 后，事件将正常发出。

### 模拟特定事件

如果您调用 `emitter.fake` 方法而不带任何参数，它将模拟所有事件。如果您想模拟特定事件，请将事件名称或类作为第一个参数传递。

```ts
// 仅模拟 user:registered 事件
emitter.fake('user:registered')

// 模拟多个事件
emitter.fake([UserRegistered, OrderUpdated])
```

多次调用 `emitter.fake` 方法将删除旧的 fake。

### 事件断言

您可以使用 `assertEmitted`、`assertNotEmitted`、`assertNoneEmitted` 和 `assertEmittedCount` 方法为模拟的事件编写断言。

`assertEmitted` 和 `assertNotEmitted` 方法接受事件名称或类构造函数作为第一个参数，以及一个可选的查找函数，该函数必须返回一个布尔值来标记事件已发出。

```ts
const events = emitter.fake()

events.assertEmitted('user:registered')
events.assertNotEmitted(OrderUpdated)
```

```ts
// title: 使用回调

events.assertEmitted(OrderUpdated, ({ data }) => {
  /**
   * 仅当 orderId 匹配时，才将事件视为已发出
   */
  return data.order.id === orderId
})
```

```ts
// title: 断言事件计数
// 断言特定事件的计数
events.assertEmittedCount(OrderUpdated, 1)

// 断言没有事件被发出
events.assertNoneEmitted()
```
