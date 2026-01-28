---
summary: 学习如何使用访问令牌守卫通过访问令牌验证 HTTP 请求。
---

# 访问令牌守卫

访问令牌在 API 上下文中验证 HTTP 请求，其中服务器无法在最终用户设备上持久化 cookie，例如第三方访问 API 或移动应用程序的身份验证。

访问令牌可以以任何格式生成；例如，符合 JWT 标准的令牌称为 JWT 访问令牌，专有格式的令牌称为不透明访问令牌。

AdonisJS 使用不透明访问令牌，其结构和存储方式如下：

- 令牌由加密安全的随机值表示，后缀为 CRC32 校验和。
- 令牌值的哈希值持久化在数据库中。此哈希值用于在身份验证时验证令牌。
- 最终令牌值经过 base64 编码，并以 `oat_` 为前缀。前缀可以自定义。
- 前缀和 CRC32 校验和后缀有助于 [秘密扫描工具](https://docs.github.com/en/code-security/secret-scanning/about-secret-scanning) 识别令牌并防止它们在代码库中泄露。

## 配置用户模型

在使用访问令牌守卫之前，您必须使用用户模型设置令牌提供程序。**令牌提供程序用于创建、列出和验证访问令牌**。

身份验证包附带一个数据库令牌提供程序，它将令牌持久化在 SQL 数据库中。您可以按如下方式配置它：

```ts
import { BaseModel } from '@adonisjs/lucid/orm'
// highlight-start
import { DbAccessTokensProvider } from '@adonisjs/auth/access_tokens'
// highlight-end

export default class User extends BaseModel {
  // ...模型属性的其余部分

  // highlight-start
  static accessTokens = DbAccessTokensProvider.forModel(User)
  // highlight-end
}
```

`DbAccessTokensProvider.forModel` 接受用户模型作为第一个参数，并接受一个选项对象作为第二个参数。

```ts
export default class User extends BaseModel {
  // ...模型属性的其余部分

  static accessTokens = DbAccessTokensProvider.forModel(User, {
    expiresIn: '30 days',
    prefix: 'oat_',
    table: 'auth_access_tokens',
    type: 'auth_token',
    tokenSecretLength: 40,
  })
}
```

<dl>

<dt>

expiresIn

</dt>

<dd>

令牌过期后的持续时间。您可以传递以秒为单位的数值或作为字符串的 [时间表达式](https://github.com/poppinss/utils?tab=readme-ov-file#secondsparseformat)。

默认情况下，令牌是长期有效的，不会过期。此外，您可以在生成令牌时指定令牌的过期时间。

</dd>

<dt>

prefix

</dt>

<dd>

公开共享令牌值的前缀。定义前缀有助于 [秘密扫描工具](https://github.blog/2021-04-05-behind-githubs-new-authentication-token-formats/#identifiable-prefixes) 识别令牌并防止它们在代码库中泄露。

在颁发令牌后更改前缀将使它们无效。因此，请仔细选择前缀，不要经常更改它们。

默认为 `oat_`。

</dd>

<dt>

table

</dt>

<dd>

用于存储访问令牌的数据库表名。默认为 `auth_access_tokens`。

</dd>

<dt>

type

</dt>

<dd>

用于标识令牌桶的唯一类型。如果您在单个应用程序中颁发多种类型的令牌，则必须为它们全部定义一个唯一的类型。

默认为 `auth_token`。

</dd>

<dt>

tokenSecretLength

</dt>

<dd>

随机令牌值的长度（以字符为单位）。默认为 `40`。

</dd>

</dl>

---

配置令牌提供程序后，您可以开始代表用户 [颁发令牌](#issuing-a-token)。您不必设置身份验证守卫来颁发令牌。守卫需要验证令牌。

## 创建访问令牌数据库表

我们在初始设置期间为 `auth_access_tokens` 表创建迁移文件。迁移文件存储在 `database/migrations` 目录中。

您可以通过执行 `migration:run` 命令来创建数据库表。

```sh
node ace migration:run
```

但是，如果您出于某种原因手动配置身份验证包，您可以手动创建迁移文件，并在其中复制粘贴以下代码片段。

```sh
node ace make:migration auth_access_tokens
```

```ts
import { BaseSchema } from '@adonisjs/lucid/schema'

export default class extends BaseSchema {
  protected tableName = 'auth_access_tokens'

  async up() {
    this.schema.createTable(this.tableName, (table) => {
      table.increments('id')
      table
        .integer('tokenable_id')
        .notNullable()
        .unsigned()
        .references('id')
        .inTable('users')
        .onDelete('CASCADE')

      table.string('type').notNullable()
      table.string('name').nullable()
      table.string('hash').notNullable()
      table.text('abilities').notNullable()
      table.timestamp('created_at')
      table.timestamp('updated_at')
      table.timestamp('last_used_at').nullable()
      table.timestamp('expires_at').nullable()
    })
  }

  async down() {
    this.schema.dropTable(this.tableName)
  }
}
```

## 颁发令牌

根据您的应用程序，您可能会在登录期间或登录后从应用程序仪表板颁发令牌。在任何一种情况下，颁发令牌都需要一个用户对象（将为其生成令牌），您可以直接使用 `用户` 模型生成它们。

如果您使用访问令牌作为用户登录/注销的主要方式，您可能更喜欢直接通过身份验证守卫创建/使令牌失效，请参阅 [登录和注销](#logging-in-and-out)。

在以下示例中，我们 **通过 id 查找用户** 并使用 `User.accessTokens.create` 方法 **向他们颁发访问令牌**。当然，在现实世界的应用程序中，您将通过身份验证保护此端点，但现在让我们保持简单。

`.create` 方法接受用户模型的实例，并返回 [AccessToken](https://github.com/adonisjs/auth/blob/main/modules/access_tokens_guard/access_token.ts) 类的实例。

`token.value` 属性包含必须与用户共享的值（包装为 [Secret](../references/helpers.md#secret)）。该值仅在生成令牌时可用，用户将无法再次看到它。

```ts
import router from '@adonisjs/core/services/router'
import User from '#models/user'

router.post('users/:id/tokens', async ({ params }) => {
  const user = await User.findOrFail(params.id)
  const token = await User.accessTokens.create(user)

  return {
    type: 'bearer',
    value: token.value!.release(),
  }
})
```

您还可以直接在响应中返回 `token`，它将序列化为以下 JSON 对象。

```ts
router.post('users/:id/tokens', async ({ params }) => {
  const user = await User.findOrFail(params.id)
  const token = await User.accessTokens.create(user)

  // delete-start
  return {
    type: 'bearer',
    value: token.value!.release(),
  }
  // delete-end
  // insert-start
  return token
  // insert-end
})

/**
 * response: {
 *   type: 'bearer',
 *   value: 'oat_MTA.aWFQUmo2WkQzd3M5cW0zeG5JeHdiaV9rOFQzUWM1aTZSR2xJaDZXYzM5MDE4MzA3NTU',
 *   expiresAt: null,
 * }
 */
```

### 定义能力

根据您正在构建的应用程序，您可能希望将访问令牌限制为仅执行特定任务。例如，颁发一个允许读取和列出项目而不创建或删除它们的令牌。

在以下示例中，我们将能力数组定义为第二个参数。能力被序列化为 JSON 字符串并持久化在数据库中。

对于身份验证包，能力没有真正的意义。您的应用程序需要在执行给定操作之前检查令牌能力。

```ts
await User.accessTokens.create(user, ['server:create', 'server:read'])
```

### 令牌能力与 Bouncer 能力

您不应将令牌能力与 [bouncer 授权检查](../security/authorization.md#defining-abilities) 混淆。让我们尝试通过一个实际示例来理解差异。

- 假设您定义了一个 **允许管理员用户创建新项目的 bouncer 能力**。

- 同一个管理员用户为自己创建了一个令牌，但为了防止令牌滥用，他们将令牌能力限制为 **读取项目**。

- 现在，在您的应用程序中，您将必须实施访问控制，该控制允许管理员用户创建新项目，同时禁止令牌创建新项目。

您可以按如下方式为此用例编写 bouncer 能力：

:::note

`user.currentAccessToken` 指的是在当前 HTTP 请求期间用于身份验证的访问令牌。您可以在 [验证请求](#the-current-access-token) 部分中了解更多信息。

:::

```ts
import { AccessToken } from '@adonisjs/auth/access_tokens'
import { Bouncer } from '@adonisjs/bouncer'

export const createProject = Bouncer.ability(
  (user: User & { currentAccessToken?: AccessToken }) => {
    /**
     * 如果没有 "currentAccessToken" 令牌属性，则意味着
     * 用户在没有访问令牌的情况下进行了身份验证
     */
    if (!user.currentAccessToken) {
      return user.isAdmin
    }

    /**
     * 否则，检查用户是否为管理员以及他们
     * 用于身份验证的令牌是否允许 "project:create"
     * 能力。
     */
    return user.isAdmin && user.currentAccessToken.allows('project:create')
  },
)
```

### 令牌过期

默认情况下，令牌是长期有效的，它们永远不会过期。但是，您可以在 [配置令牌提供程序](#configuring-the-user-model) 时或生成令牌时定义过期时间。

过期时间可以定义为表示秒数的数值或基于字符串的时间表达式。

```ts
await User.accessTokens.create(
  user, // 对于用户
  ['*'], // 具有所有能力
  {
    expiresIn: '30 days', // 30 天后过期
  },
)
```

### 命名令牌

默认情况下，令牌没有名称。但是，您可以在生成令牌时为它们分配一个名称。例如，如果您允许应用程序的用户自生成令牌，您还可以要求他们指定一个可识别的名称。

```ts
await User.accessTokens.create(user, ['*'], {
  name: request.input('token_name'),
  expiresIn: '30 days',
})
```

## 配置守卫

现在我们可以颁发令牌了，让我们配置一个身份验证守卫来验证请求并对用户进行身份验证。必须在 `guards` 对象下的 `config/auth.ts` 文件中配置守卫。

```ts
// title: config/auth.ts
import { defineConfig } from '@adonisjs/auth'
// highlight-start
import { tokensGuard, tokensUserProvider } from '@adonisjs/auth/access_tokens'
// highlight-end

const authConfig = defineConfig({
  default: 'api',
  guards: {
    // highlight-start
    api: tokensGuard({
      provider: tokensUserProvider({
        tokens: 'accessTokens',
        model: () => import('#models/user'),
      }),
    }),
    // highlight-end
  },
})

export default authConfig
```

`tokensGuard` 方法创建 [AccessTokensGuard](https://github.com/adonisjs/auth/blob/main/modules/access_tokens_guard/guard.ts) 类的实例。它接受一个用户提供程序，该提供程序可用于验证令牌和查找用户。

`tokensUserProvider` 方法接受以下选项，并返回 [AccessTokensLucidUserProvider](https://github.com/adonisjs/auth/blob/main/modules/access_tokens_guard/user_providers/lucid.ts) 类的实例。

- `model`：用于查找用户的 Lucid 模型。
- `tokens`：模型的静态属性名称，用于引用令牌提供程序。

## 验证请求

配置守卫后，您可以开始使用 `auth` 中间件验证请求，或手动调用 `auth.authenticate` 方法。

`auth.authenticate` 方法返回经过身份验证的用户的用户模型实例，或者在无法验证请求时抛出 [E_UNAUTHORIZED_ACCESS](../references/exceptions.md#e_unauthorized_access) 异常。

```ts
import router from '@adonisjs/core/services/router'

router.post('projects', async ({ auth }) => {
  // 使用默认守卫进行身份验证
  const user = await auth.authenticate()

  // 使用命名守卫进行身份验证
  const user = await auth.authenticateUsing(['api'])
})
```

### 使用 auth 中间件

您可以不用手动调用 `authenticate` 方法。您可以使用 `auth` 中间件来验证请求或抛出异常。

auth 中间件接受一个守卫数组，用于验证请求。在一个提到的守卫验证请求后，身份验证过程停止。

```ts
import router from '@adonisjs/core/services/router'
import { middleware } from '#start/kernel'

router
  .post('projects', async ({ auth }) => {
    console.log(auth.user) // 用户
    console.log(auth.authenticatedViaGuard) // 'api'
    console.log(auth.user!.currentAccessToken) // 访问令牌
  })
  .use(
    middleware.auth({
      guards: ['api'],
    }),
  )
```

### 检查请求是否已通过身份验证

您可以使用 `auth.isAuthenticated` 标志检查请求是否已通过身份验证。对于经过身份验证的请求，`auth.user` 的值将始终被定义。

```ts
import { HttpContext } from '@adonisjs/core/http'

class PostsController {
  async store({ auth }: HttpContext) {
    if (auth.isAuthenticated) {
      await auth.user!.related('posts').create(postData)
    }
  }
}
```

### 获取经过身份验证的用户或失败

如果您不喜欢在 `auth.user` 属性上使用 [非空断言运算符](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#non-null-assertion-operator-postfix-)，您可以使用 `auth.getUserOrFail` 方法。此方法将返回用户对象或抛出 [E_UNAUTHORIZED_ACCESS](../references/exceptions.md#e_unauthorized_access) 异常。

```ts
import { HttpContext } from '@adonisjs/core/http'

class PostsController {
  async store({ auth }: HttpContext) {
    const user = auth.getUserOrFail()
    await user.related('posts').create(postData)
  }
}
```

## 当前访问令牌

访问令牌守卫在成功验证请求后在用户对象上定义 `currentAccessToken` 属性。`currentAccessToken` 属性是 [AccessToken](https://github.com/adonisjs/auth/blob/main/modules/access_tokens_guard/access_token.ts) 类的实例。

您可以使用 `currentAccessToken` 对象来获取令牌的能力或检查令牌的过期时间。此外，在身份验证期间，守卫将更新 `last_used_at` 列以反映当前时间戳。

如果您在代码库的其余部分中使用 `currentAccessToken` 作为类型引用用户模型，您可能希望自己在模型上声明此属性。

:::caption{for="error"}

**而不是合并 `currentAccessToken`**

:::

```ts
import { AccessToken } from '@adonisjs/auth/access_tokens'

Bouncer.ability((user: User & { currentAccessToken?: AccessToken }) => {})
```

:::caption{for="success"}

**将其声明为模型上的属性**

:::

```ts
import { AccessToken } from '@adonisjs/auth/access_tokens'

export default class User extends BaseModel {
  currentAccessToken?: AccessToken
}
```

```ts
Bouncer.ability((user: User) => {})
```

## 列出所有令牌

您可以使用令牌提供程序通过 `accessTokens.all` 方法获取所有令牌的列表。返回值将是 `AccessToken` 类实例的数组。

```ts
router
  .get('/tokens', async ({ auth }) => {
    return User.accessTokens.all(auth.user!)
  })
  .use(
    middleware.auth({
      guards: ['api'],
    }),
  )
```

`all` 方法还返回过期的令牌。您可能希望在呈现列表之前过滤它们，或者在令牌旁边显示 **"令牌已过期"** 消息。例如

```edge
@each(token in tokens)
  <h2> {{ token.name }} </h2>
  @if(token.isExpired())
    <p> 已过期 </p>
  @end

  <p> 能力: {{ token.abilities.join(',') }} </p>
@end
```

## 删除令牌

您可以使用 `accessTokens.delete` 方法删除令牌。该方法接受用户作为第一个参数，令牌 id 作为第二个参数。

```ts
await User.accessTokens.delete(user, token.identifier)
```

## 事件

请查看 [事件参考指南](../references/events.md#access_tokens_authauthentication_attempted) 以查看访问令牌守卫发出的可用事件列表。

## 登录和注销

访问令牌有时是将用户登录和注销的首选方法 - 例如，在对本机应用程序进行身份验证时。

为了适应这些情况，访问令牌守卫提供了一个类似于 [会话守卫](./session_guard.md) 的 [登录](./session_guard.md#performing-login) 和 [注销](./session_guard.md#performing-logout) 方法的 API。

登录：

```ts
const token = await auth.use('api').createToken(user)
```

注销（当前经过身份验证的令牌）：

```ts
await auth.use('api').invalidateToken()
```

### 会话控制器示例

假设访问令牌守卫 `api` 已经到位（例如，您已经设置了：[用户模型](#configuring-the-user-model)、[访问令牌](#creating-the-access-tokens-database-table) 和 [身份验证守卫](#configuring-the-guard)），会话控制器可以通过以下方式实现：

```ts
// title: app/controllers/session_controller.ts
import User from '#models/user'
import { HttpContext } from '@adonisjs/core/http'

export default class SessionController {
  async store({ request, auth, response }: HttpContext) {
    const { email, password } = request.only(['email', 'password'])
    const user = await User.verifyCredentials(email, password)

    return await auth.use('api').createToken(user)
  }

  async destroy({ request, auth, response }: HttpContext) {
    await auth.use('api').invalidateToken()
  }
}
```

```ts
// title: start/routes.ts
import router from '@adonisjs/core/services/router'

const SessionController = () => import('#controllers/session_controller')

router.post('session', [SessionController, 'store'])
router
  .delete('session', [SessionController, 'destroy'])
  .use(middleware.auth({ guards: ['api'] }))
```

:::warning

当 `User.verifyCredentials` 失败（并抛出 [E_INVALID_CREDENTIALS](../references/exceptions#e_invalid_credentials)）时，使用 [内容协商](../authentication/verifying_user_credentials.md#handling-exceptions) 获取适当的响应。

在上述示例的情况下，客户端应在 post 请求中包含 `Accept=application/json` 标头到 `/session`。这可确保失败将导致 json 格式的响应，而不是重定向。

:::

:::tip

如果您使用访问令牌从外部源（如移动应用程序）登录，您可能需要禁用 [CSRF 保护](../security/securing_ssr_applications.md#csrf-protection)。

您可以全局禁用 CSRF 保护（如果您的应用程序仅用作 API），或者为 API 路由（包括 `/session` 路由）添加例外。请参阅 [shield 配置参考](https://docs.adonisjs.com/guides/security/securing-ssr-applications#config-reference)

:::
