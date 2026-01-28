---
summary: 学习如何在 AdonisJS 测试期间模拟或伪造依赖项。
---

# 模拟和伪造

在测试应用程序时，你可能希望模拟或伪造特定依赖项以防止实际实现运行。例如，你希望在运行测试时避免向客户发送电子邮件，也不调用第三方支付网关等服务。

AdonisJS 提供了几种不同的 API 和建议，你可以使用这些 API 和建议来伪造、模拟或存根依赖项。

## 使用伪造 API

伪造是专门为测试创建的具有工作实现的对象。例如，AdonisJS 的邮件程序模块有一个伪造实现，你可以在测试期间使用它来在内存中收集外发邮件并编写断言。

我们为以下容器服务提供伪造实现。伪造 API 与模块文档一起记录。

- [Emitter 伪造](../digging_deeper/emitter.md#faking-events-during-tests)
- [Hash 伪造](../security/hashing.md#faking-hash-service-during-tests)
- [伪造邮件程序](../digging_deeper/mail.md#fake-mailer)
- [Drive 伪造](../digging_deeper/drive.md#faking-disks)

## 依赖注入和伪造

如果你在应用程序中使用依赖注入，或使用[容器创建类实例](../concepts/dependency_injection.md)，你可以使用 `container.swap` 方法为类提供伪造实现。

在以下示例中，我们将 `UserService` 注入到 `UsersController` 中。

```ts
import UserService from '#services/user_service'
import { inject } from '@adonisjs/core'

export default class UsersController {
  @inject()
  index(service: UserService) {}
}
```

在测试期间，我们可以按如下方式提供 `UserService` 类的备用/伪造实现。

```ts
import UserService from '#services/user_service'
import app from '@adonisjs/core/services/app'

test('获取所有用户', async () => {
  // highlight-start
  class FakeService extends UserService {
    all() {
      return [{ id: 1, username: 'virk' }]
    }
  }

  /**
   * 将 `UserService` 与
   * `FakeService` 的实例交换
   */
  app.container.swap(UserService, () => {
    return new FakeService()
  })

  /**
   * 测试逻辑放在这里
   */
  // highlight-end
})
```

测试完成后，你必须使用 `container.restore` 方法恢复伪造。

```ts
app.container.restore(UserService)

// 恢复 UserService 和 PostService
app.container.restoreAll([UserService, PostService])

// 恢复全部
app.container.restoreAll()
```

## 使用 Sinon.js 进行模拟和存根

[Sinon](https://sinonjs.org/#get-started) 是 Node.js 生态系统中一个成熟、久经考验的模拟库。如果你经常使用模拟和存根，我们建议使用 Sinon，因为它与 TypeScript 配合得很好。

## 模拟网络请求

如果你的应用程序向第三方服务发出外发 HTTP 请求，你可以在测试期间使用 [nock](https://github.com/nock/nock) 来模拟网络请求。

由于 nock 拦截来自 [Node.js HTTP 模块](https://nodejs.org/dist/latest-v20.x/docs/api/http.html#class-httpclientrequest) 的所有外发请求，因此它几乎适用于每个第三方库，如 `got`、`axios` 等。

## 冻结时间

你可以使用 [timekeeper](https://www.npmjs.com/package/timekeeper) 包在编写测试时冻结或穿越时间。timekeeper 包通过模拟 `Date` 类来工作。

在以下示例中，我们将 `timekeeper` 的 API 封装在 [Japa 测试资源](https://japa.dev/docs/test-resources) 中。

```ts
import { getActiveTest } from '@japa/runner'
import timekeeper from 'timekeeper'

export function timeTravel(secondsToTravel: number) {
  const test = getActiveTest()
  if (!test) {
    throw new Error('无法在 Japa 测试之外使用 "timeTravel"')
  }

  timekeeper.reset()

  const date = new Date()
  date.setSeconds(date.getSeconds() + secondsToTravel)
  timekeeper.travel(date)

  test.cleanup(() => {
    timekeeper.reset()
  })
}
```

你可以按如下方式在测试中使用 `timeTravel` 方法。

```ts
import { timeTravel } from '#test_helpers'

test('2 小时后过期令牌', async ({ assert }) => {
  /**
   * 穿越到未来 3 小时
   */
  timeTravel(60 * 60 * 3)
})
```
