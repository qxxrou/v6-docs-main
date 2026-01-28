---
summary: 学习如何为 AdonisJS 创建自定义认证守卫。
---

# 创建自定义认证守卫

auth 包允许你创建自定义认证守卫，用于内置守卫无法满足的用例。在本指南中，我们将创建一个使用 JWT 令牌进行身份验证的守卫。

认证守卫围绕以下概念展开：

- **用户提供者（User Provider）**：守卫必须是用户无关的。它们不应该硬编码从数据库查询和查找用户的函数。相反，守卫应该依赖于用户提供者，并将其实现作为构造函数依赖项接受。

- **守卫实现**：守卫实现必须遵守 `GuardContract` 接口。该接口描述了将守卫与 Auth 层的其余部分集成所需的 API。

## 创建 `UserProvider` 接口

守卫负责定义 `UserProvider` 接口及其应包含的方法/属性。例如，[会话守卫（Session guard）] 接受的用户提供者比 [访问令牌守卫（Access tokens guard）] 接受的用户提供者简单得多。

因此，无需创建满足每个守卫实现的用户提供者。每个守卫都可以规定它们接受的用户提供者的要求。

对于本示例，我们需要一个提供者来使用 `user ID` 查找数据库中的用户。我们不关心使用哪个数据库或如何执行查询。这是实现用户提供者的开发人员的责任。

:::note

我们将在本指南中编写的所有代码最初都可以存储在 `app/auth/guards` 目录中的单个文件中。

:::

```ts
// title: app/auth/guards/jwt.ts
import { symbols } from '@adonisjs/auth'

/**
 * 用户提供者和守卫之间的桥梁
 */
export type JwtGuardUser<RealUser> = {
  /**
   * 返回用户的唯一 ID
   */
  getId(): string | number | BigInt

  /**
   * 返回原始用户对象
   */
  getOriginal(): RealUser
}

/**
 * JWT 守卫接受的用户提供者的接口
 */
export interface JwtUserProviderContract<RealUser> {
  /**
   * 守卫实现可用于推断实际用户数据类型的属性
   */
  [symbols.PROVIDER_REAL_USER]: RealUser

  /**
   * 创建一个用户对象，充当守卫和真实用户值之间的适配器
   */
  createUserForGuard(user: RealUser): Promise<JwtGuardUser<RealUser>>

  /**
   * 通过 ID 查找用户
   */
  findById(
    identifier: string | number | BigInt,
  ): Promise<JwtGuardUser<RealUser> | null>
}
```

在上面的示例中，`JwtUserProviderContract` 接口接受一个名为 `RealUser` 的通用用户属性。由于此接口不知道实际用户（我们从数据库获取的用户）是什么样子，因此它将其作为通用参数接受。例如：

- 使用 Lucid 模型的实现将返回 Model 的实例。因此，`RealUser` 的值将是该实例。

- 使用 Prisma 的实现将返回具有特定属性的用户对象；因此，`RealUser` 的值将是该对象。

总而言之，`JwtUserProviderContract` 将其留给用户提供者实现来决定用户的数据类型。

### 理解 `JwtGuardUser` 类型

`JwtGuardUser` 类型充当用户提供者和守卫之间的桥梁。守卫使用 `getId` 方法获取用户的唯一 ID，并在验证请求后使用 `getOriginal` 方法获取用户对象。

## 实现守卫

让我们创建 `JwtGuard` 类并定义 [`GuardContract`] 接口所需的方法/属性。最初，此文件中会有很多错误，但这没关系；随着我们的进展，所有错误都会消失。

:::note

请花一些时间阅读以下示例中每个属性/方法旁边的注释。

:::

```ts
import { symbols } from '@adonisjs/auth'
import { AuthClientResponse, GuardContract } from '@adonisjs/auth/types'

export class JwtGuard<
  UserProvider extends JwtUserProviderContract<unknown>,
> implements GuardContract<UserProvider[typeof symbols.PROVIDER_REAL_USER]> {
  /**
   * 守卫发出的事件列表及其类型
   */
  declare [symbols.GUARD_KNOWN_EVENTS]: {}

  /**
   * 守卫驱动程序的唯一名称
   */
  driverName: 'jwt' = 'jwt'

  /**
   * 一个标志，用于了解在当前 HTTP 请求期间是否尝试了身份验证
   */
  authenticationAttempted: boolean = false

  /**
   * 一个布尔值，用于了解当前请求是否已通过身份验证
   */
  isAuthenticated: boolean = false

  /**
   * 对当前已通过身份验证的用户的引用
   */
  user?: UserProvider[typeof symbols.PROVIDER_REAL_USER]

  /**
   * 为给定用户生成 JWT 令牌
   */
  async generate(user: UserProvider[typeof symbols.PROVIDER_REAL_USER]) {}

  /**
   * 验证当前 HTTP 请求并返回用户实例（如果存在有效的 JWT 令牌）
   * 或抛出异常
   */
  async authenticate(): Promise<
    UserProvider[typeof symbols.PROVIDER_REAL_USER]
  > {}

  /**
   * 与 authenticate 相同，但不抛出异常
   */
  async check(): Promise<boolean> {}

  /**
   * 返回已通过身份验证的用户或抛出错误
   */
  getUserOrFail(): UserProvider[typeof symbols.PROVIDER_REAL_USER] {}

  /**
   * 在测试期间使用 "loginAs" 方法登录用户时，Japa 会调用此方法
   */
  async authenticateAsClient(
    user: UserProvider[typeof symbols.PROVIDER_REAL_USER],
  ): Promise<AuthClientResponse> {}
}
```

## 接受用户提供者

守卫必须接受用户提供者在身份验证期间查找用户。你可以将其作为构造函数参数接受并存储私有引用。

```ts
export class JwtGuard<
  UserProvider extends JwtUserProviderContract<unknown>,
> implements GuardContract<UserProvider[typeof symbols.PROVIDER_REAL_USER]> {
  // insert-start
  #userProvider: UserProvider

  constructor(userProvider: UserProvider) {
    this.#userProvider = userProvider
  }
  // insert-end
}
```

## 生成令牌

让我们实现 `generate` 方法并为给定用户创建令牌。我们将安装并使用 npm 中的 `jsonwebtoken` 包来生成令牌。

```sh
npm i jsonwebtoken @types/jsonwebtoken
```

此外，我们还必须使用**密钥**来签署令牌，因此让我们更新 `constructor` 方法并通过 options 对象接受密钥作为选项。

```ts
// insert-start
import jwt from 'jsonwebtoken'

export type JwtGuardOptions = {
  secret: string
}
// insert-end

export class JwtGuard<UserProvider extends JwtUserProviderContract<unknown>>
  implements GuardContract<UserProvider[typeof symbols.PROVIDER_REAL_USER]>
{
  #userProvider: UserProvider
  // insert-start
  #options: JwtGuardOptions
  // insert-end

  constructor(
    userProvider: UserProvider
    // insert-start
    options: JwtGuardOptions
    // insert-end
  ) {
    this.#userProvider = userProvider
    // insert-start
    this.#options = options
    // insert-end
  }

  /**
   * 为给定用户生成 JWT 令牌
   */
  async generate(
    user: UserProvider[typeof symbols.PROVIDER_REAL_USER]
  ) {
    // insert-start
    const providerUser = await this.#userProvider.createUserForGuard(user)
    const token = jwt.sign({ userId: providerUser.getId() }, this.#options.secret)

    return {
      type: 'bearer',
      token: token
    }
    // insert-end
  }
}
```

- 首先，我们使用 `userProvider.createUserForGuard` 方法创建提供者用户实例（又名真实用户和守卫之间的桥梁）。

- 接下来，我们使用 `jwt.sign` 方法创建带有 `userId` 的有效负载的签名令牌并返回它。

## 验证请求

验证请求包括：

- 从请求头或 cookie 读取 JWT 令牌
- 验证其真实性
- 获取为其生成令牌的用户

我们的守卫需要访问 [HttpContext] 来读取请求头和 cookie，因此让我们更新类 `constructor` 并将其作为参数接受。

```ts
// insert-start
import type { HttpContext } from '@adonisjs/core/http'
// insert-end

export class JwtGuard<
  UserProvider extends JwtUserProviderContract<unknown>,
> implements GuardContract<UserProvider[typeof symbols.PROVIDER_REAL_USER]> {
  // insert-start
  #ctx: HttpContext
  // insert-end
  #userProvider: UserProvider
  #options: JwtGuardOptions

  constructor(
    // insert-start
    ctx: HttpContext,
    // insert-end
    userProvider: UserProvider,
    options: JwtGuardOptions,
  ) {
    // insert-start
    this.#ctx = ctx
    // insert-end
    this.#userProvider = userProvider
    this.#options = options
  }
}
```

对于此示例，我们将从 `authorization` 头读取令牌。但是，你可以调整实现以支持 cookie。

```ts
import {
  symbols,
  // insert-start
  errors,
  // insert-end
} from '@adonisjs/auth'

export class JwtGuard<
  UserProvider extends JwtUserProviderContract<unknown>,
> implements GuardContract<UserProvider[typeof symbols.PROVIDER_REAL_USER]> {
  /**
   * 验证当前 HTTP 请求并返回用户实例（如果存在有效的 JWT 令牌）
   * 或抛出异常
   */
  async authenticate(): Promise<
    UserProvider[typeof symbols.PROVIDER_REAL_USER]
  > {
    /**
     * 当已经为给定请求完成时，避免重新验证
     */
    if (this.authenticationAttempted) {
      return this.getUserOrFail()
    }
    this.authenticationAttempted = true

    /**
     * 确保认证头存在
     */
    const authHeader = this.#ctx.request.header('authorization')
    if (!authHeader) {
      throw new errors.E_UNAUTHORIZED_ACCESS('Unauthorized access', {
        guardDriverName: this.driverName,
      })
    }

    /**
     * 分割头值并从中读取令牌
     */
    const [, token] = authHeader.split('Bearer ')
    if (!token) {
      throw new errors.E_UNAUTHORIZED_ACCESS('Unauthorized access', {
        guardDriverName: this.driverName,
      })
    }

    /**
     * 验证令牌
     */
    const payload = jwt.verify(token, this.#options.secret)
    if (typeof payload !== 'object' || !('userId' in payload)) {
      throw new errors.E_UNAUTHORIZED_ACCESS('Unauthorized access', {
        guardDriverName: this.driverName,
      })
    }

    /**
     * 按用户 ID 获取用户并保存对其的引用
     */
    const providerUser = await this.#userProvider.findById(payload.userId)
    if (!providerUser) {
      throw new errors.E_UNAUTHORIZED_ACCESS('Unauthorized access', {
        guardDriverName: this.driverName,
      })
    }

    this.user = providerUser.getOriginal()
    return this.getUserOrFail()
  }
}
```

## 实现 `check` 方法

`check` 方法是 `authenticate` 方法的静默版本，你可以按如下方式实现它。

```ts
export class JwtGuard<
  UserProvider extends JwtUserProviderContract<unknown>,
> implements GuardContract<UserProvider[typeof symbols.PROVIDER_REAL_USER]> {
  /**
   * 与 authenticate 相同，但不抛出异常
   */
  async check(): Promise<boolean> {
    // insert-start
    try {
      await this.authenticate()
      return true
    } catch {
      return false
    }
    // insert-end
  }
}
```

## 实现 `getUserOrFail` 方法

最后，让我们实现 `getUserOrFail` 方法。它应返回用户实例或抛出错误（如果用户不存在）。

```ts
export class JwtGuard<
  UserProvider extends JwtUserProviderContract<unknown>,
> implements GuardContract<UserProvider[typeof symbols.PROVIDER_REAL_USER]> {
  /**
   * 返回已通过身份验证的用户或抛出错误
   */
  getUserOrFail(): UserProvider[typeof symbols.PROVIDER_REAL_USER] {
    // insert-start
    if (!this.user) {
      throw new errors.E_UNAUTHORIZED_ACCESS('Unauthorized access', {
        guardDriverName: this.driverName,
      })
    }

    return this.user
    // insert-end
  }
}
```

## 实现 `authenticateAsClient` 方法

`authenticateAsClient` 方法在测试期间使用，当你想在测试期间通过 [`loginAs` 方法] 登录用户时使用。对于 JWT 实现，此方法应返回包含 JWT 令牌的 `authorization` 头。

```ts
export class JwtGuard<
  UserProvider extends JwtUserProviderContract<unknown>,
> implements GuardContract<UserProvider[typeof symbols.PROVIDER_REAL_USER]> {
  /**
   * 在测试期间使用 "loginAs" 方法登录用户时，Japa 会调用此方法
   */
  async authenticateAsClient(
    user: UserProvider[typeof symbols.PROVIDER_REAL_USER],
  ): Promise<AuthClientResponse> {
    // insert-start
    const token = await this.generate(user)
    return {
      headers: {
        authorization: `Bearer ${token.token}`,
      },
    }
    // insert-end
  }
}
```

## 使用守卫

让我们前往 `config/auth.ts` 并在 `guards` 列表中注册守卫。

```ts
import { defineConfig } from '@adonisjs/auth'
// insert-start
import { sessionUserProvider } from '@adonisjs/auth/session'
import env from '#start/env'
import { JwtGuard } from '../app/auth/jwt/guard.js'
// insert-end

// insert-start
const jwtConfig = {
  secret: env.get('APP_KEY'),
}
const userProvider = sessionUserProvider({
  model: () => import('#models/user'),
})
// insert-end

const authConfig = defineConfig({
  default: 'jwt',
  guards: {
    // insert-start
    jwt: (ctx) => {
      return new JwtGuard(ctx, userProvider, jwtConfig)
    },
    // insert-end
  },
})

export default authConfig
```

正如你所注意到的，我们正在将 `sessionUserProvider` 与我们的 `JwtGuard` 实现一起使用。这是因为 `JwtUserProviderContract` 接口与会话守卫创建的用户提供者兼容。

因此，我们不是创建自己的用户提供者实现，而是重用会话守卫中的用户提供者。

## 最终示例

实现完成后，你可以像其他内置守卫一样使用 `jwt` 守卫。以下是有关如何生成和验证 JWT 令牌的示例。

```ts
import User from '#models/user'
import router from '@adonisjs/core/services/router'
import { middleware } from './kernel.js'

router.post('login', async ({ request, auth }) => {
  const { email, password } = request.all()
  const user = await User.verifyCredentials(email, password)

  return await auth.use('jwt').generate(user)
})

router
  .get('/', async ({ auth }) => {
    return auth.getUserOrFail()
  })
  .use(middleware.auth())
```
