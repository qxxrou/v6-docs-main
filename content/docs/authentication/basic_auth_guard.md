---
summary: 学习如何使用基本认证守卫通过 HTTP 认证框架对用户进行身份验证。
---

# 基本认证守卫

基本认证守卫是 [HTTP 认证框架](https://developer.mozilla.org/en-US/docs/Web/HTTP/Authentication) 的一个实现，客户端必须通过 `Authorization` 头传递 base64 编码的用户凭据字符串。如果凭据有效，服务器允许请求。否则，会显示一个原生的 Web 提示框要求重新输入凭据。

## 配置守卫

认证守卫在 `config/auth.ts` 文件中定义。你可以在该文件的 `guards` 对象下配置多个守卫。

```ts
import { defineConfig } from '@adonisjs/auth'
// highlight-start
import {
  basicAuthGuard,
  basicAuthUserProvider,
} from '@adonisjs/auth/basic_auth'
// highlight-end

const authConfig = defineConfig({
  default: 'basicAuth',
  guards: {
    // highlight-start
    basicAuth: basicAuthGuard({
      provider: basicAuthUserProvider({
        model: () => import('#models/user'),
      }),
    }),
    // highlight-end
  },
})

export default authConfig
```

`basicAuthGuard` 方法创建 [BasicAuthGuard](https://github.com/adonisjs/auth/blob/main/modules/basic_auth_guard/guard.ts) 类的实例。它接受一个用户提供程序，该提供程序可用于在认证期间查找用户。

`basicAuthUserProvider` 方法创建 [BasicAuthLucidUserProvider](https://github.com/adonisjs/auth/blob/main/modules/basic_auth_guard/user_providers/lucid.ts) 类的实例。它接受对用于验证用户凭据的模型的引用。

## 准备用户模型

使用 `basicAuthUserProvider` 配置的模型（本例中为 `User` 模型）必须使用 [AuthFinder](./verifying_user_credentials.md#using-the-auth-finder-mixin) 混合器在认证期间验证用户凭据。

```ts
import { DateTime } from 'luxon'
import { compose } from '@adonisjs/core/helpers'
import { BaseModel, column } from '@adonisjs/lucid/orm'
// highlight-start
import hash from '@adonisjs/core/services/hash'
import { withAuthFinder } from '@adonisjs/auth/mixins/lucid'
// highlight-end

// highlight-start
const AuthFinder = withAuthFinder(() => hash.use('scrypt'), {
  uids: ['email'],
  passwordColumnName: 'password',
})
// highlight-end

// highlight-start
export default class User extends compose(BaseModel, AuthFinder) {
  // highlight-end
  @column({ isPrimary: true })
  declare id: number

  @column()
  declare fullName: string | null

  @column()
  declare email: string

  @column()
  declare password: string

  @column.dateTime({ autoCreate: true })
  declare createdAt: DateTime

  @column.dateTime({ autoCreate: true, autoUpdate: true })
  declare updatedAt: DateTime
}
```

## 保护路由

配置好守卫后，你可以使用 `auth` 中间件来保护路由免受未经身份验证的请求。该中间件在 `start/kernel.ts` 文件的命名中间件集合下注册。

```ts
import router from '@adonisjs/core/services/router'

export const middleware = router.named({
  auth: () => import('#middleware/auth_middleware'),
})
```

```ts
// highlight-start
import { middleware } from '#start/kernel'
// highlight-end
import router from '@adonisjs/core/services/router'

router
  .get('dashboard', ({ auth }) => {
    return auth.user
  })
  .use(
    middleware.auth({
      // highlight-start
      guards: ['basicAuth'],
      // highlight-end
    }),
  )
```

### 处理认证异常

如果用户未经身份验证，auth 中间件会抛出 [E_UNAUTHORIZED_ACCESS](https://github.com/adonisjs/auth/blob/main/src/errors.ts#L21) 异常。该异常会自动转换为 HTTP 响应，并在响应中包含 [WWW-Authenticate](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/WWW-Authenticate) 头。`WWW-Authenticate` 挑战认证并触发重新输入凭据的原生 Web 提示框。

## 获取已认证用户的访问权限

你可以使用 `auth.user` 属性访问已登录的用户实例。由于你正在使用 `auth` 中间件，`auth.user` 属性将始终可用。

```ts
import { middleware } from '#start/kernel'
import router from '@adonisjs/core/services/router'

router
  .get('dashboard', ({ auth }) => {
    return `您已通过身份验证为 ${auth.user!.email}`
  })
  .use(
    middleware.auth({
      guards: ['basicAuth'],
    }),
  )
```

### 获取已认证用户或失败

如果你不喜欢在 `auth.user` 属性上使用 [非空断言操作符](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#non-null-assertion-operator-postfix-)，你可以使用 `auth.getUserOrFail` 方法。该方法将返回用户对象或抛出 [E_UNAUTHORIZED_ACCESS](../references/exceptions.md#e_unauthorized_access) 异常。

```ts
import { middleware } from '#start/kernel'
import router from '@adonisjs/core/services/router'

router
  .get('dashboard', ({ auth }) => {
    // highlight-start
    const user = auth.getUserOrFail()
    return `您已通过身份验证为 ${user.email}`
    // highlight-end
  })
  .use(
    middleware.auth({
      guards: ['basicAuth'],
    }),
  )
```
