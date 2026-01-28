---
summary: 在您的 AdonisJS 应用程序中使用 `@adonisjs/ally` 包实现社交认证。
---

# 社交认证

您可以使用 `@adonisjs/ally` 包在您的 AdonisJS 应用程序中实现社交认证。
Ally 提供了以下内置驱动程序，以及一个可扩展的 API 来[注册自定义驱动程序](#creating-a-custom-social-driver)。

- Twitter
- Facebook
- Spotify
- Google
- GitHub
- Discord
- LinkedIn

Ally 不会代表您存储任何用户或访问令牌。它实现了 OAuth2 和 OAuth1 协议，通过社交服务验证用户身份并提供用户详细信息。您可以将这些信息存储在数据库中，并使用 [auth](./introduction.md) 包在您的应用程序中登录用户。

## 安装

使用以下命令安装和配置包：

```sh
node ace add @adonisjs/ally

# 定义提供者作为 CLI 标志
node ace add @adonisjs/ally --providers=github --providers=google
```

:::disclosure{title="查看 add 命令执行的步骤"}

1. 使用检测到的包管理器安装 `@adonisjs/ally` 包。

2. 在 `adonisrc.ts` 文件中注册以下服务提供者。

   ```ts
   {
     providers: [
       // ...其他提供者
       () => import('@adonisjs/ally/ally_provider'),
     ]
   }
   ```

3. 创建 `config/ally.ts` 文件。此文件包含所选 OAuth 提供者的配置设置。

4. 定义环境变量以存储所选 OAuth 提供者的 `CLIENT_ID` 和 `CLIENT_SECRET`。

:::

## 配置

`@adonisjs/ally` 包配置存储在 `config/ally.ts` 文件中。您可以在单个配置文件中定义多个服务的配置。

另请参阅：[配置存根](https://github.com/adonisjs/ally/blob/main/stubs/config/ally.stub)

```ts
import { defineConfig, services } from '@adonisjs/ally'

defineConfig({
  github: services.github({
    clientId: env.get('GITHUB_CLIENT_ID')!,
    clientSecret: env.get('GITHUB_CLIENT_SECRET')!,
    callbackUrl: '',
  }),
  twitter: services.twitter({
    clientId: env.get('TWITTER_CLIENT_ID')!,
    clientSecret: env.get('TWITTER_CLIENT_SECRET')!,
    callbackUrl: '',
  }),
})
```

### 配置回调 URL

OAuth 提供者要求您注册一个回调 URL，以便在用户授权登录请求后处理重定向响应。

回调 URL 必须在 OAuth 服务提供者处注册。例如：如果您使用 GitHub，您必须登录到您的 GitHub 账户，[创建一个新应用](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/creating-an-oauth-app) 并使用 GitHub 界面定义回调 URL。

此外，您必须在 `config/ally.ts` 文件中使用 `callbackUrl` 属性注册相同的回调 URL。

## 使用

配置包后，您可以使用 `ctx.ally` 属性与 Ally API 交互。您可以使用 `ally.use()` 方法在配置的认证提供者之间切换。例如：

```ts
router.get('/github/redirect', ({ ally }) => {
  // GitHub 驱动程序实例
  const gh = ally.use('github')
})

router.get('/twitter/redirect', ({ ally }) => {
  // Twitter 驱动程序实例
  const twitter = ally.use('twitter')
})

// 您也可以动态检索驱动程序
router
  .get('/:provider/redirect', ({ ally, params }) => {
    const driverInstance = ally.use(params.provider)
  })
  .where('provider', /github|twitter/)
```

### 重定向用户进行认证

社交认证的第一步是将用户重定向到 OAuth 服务，并等待他们批准或拒绝认证请求。

您可以使用 `.redirect()` 方法执行重定向。

```ts
router.get('/github/redirect', ({ ally }) => {
  return ally.use('github').redirect()
})
```

您可以传递一个回调函数来在重定向期间定义自定义作用域或查询字符串值。

```ts
router.get('/github/redirect', ({ ally }) => {
  return ally.use('github').redirect((request) => {
    // highlight-start
    request.scopes(['user:email', 'repo:invite'])
    request.param('allow_signup', false)
    // highlight-end
  })
})
```

### 处理回调响应

用户批准或拒绝认证请求后，将被重定向回您的应用程序的 `callbackUrl`。

在此路由中，您可以调用 `.user()` 方法获取已登录用户的详细信息和访问令牌。但是，您还必须检查响应中可能的错误状态。

```ts
router.get('/github/callback', async ({ ally }) => {
  const gh = ally.use('github')

  /**
   * 用户通过取消登录流程拒绝了访问
   */
  if (gh.accessDenied()) {
    return '您已取消登录过程'
  }

  /**
   * OAuth 状态验证失败。这发生在
   * CSRF cookie 过期时。
   */
  if (gh.stateMisMatch()) {
    return '我们无法验证请求。请重试'
  }

  /**
   * GitHub 响应了一些错误
   */
  if (gh.hasError()) {
    return gh.getError()
  }

  /**
   * 访问用户信息
   */
  const user = await gh.user()
  return user
})
```

## 用户属性

以下是从 `.user()` 方法调用的返回值中可以访问的属性列表。这些属性在所有底层驱动程序中都是一致的。

```ts
const user = await gh.user()

user.id
user.email
user.emailVerificationState
user.name
user.nickName
user.avatarUrl
user.token
user.original
```

### id

OAuth 提供者返回的唯一 ID。

### email

OAuth 提供者返回的电子邮件地址。如果 OAuth 请求没有要求用户的电子邮件地址，该值将为 `null`。

### emailVerificationState

许多 OAuth 提供者允许未验证电子邮件的用户登录并验证 OAuth 请求。您应该使用此标志确保只有具有已验证电子邮件的用户才能登录。

以下是可能的值列表。

- `verified`：用户的电子邮件地址已通过 OAuth 提供者验证。
- `unverified`：用户的电子邮件地址未验证。
- `unsupported`：OAuth 提供者不共享电子邮件验证状态。

### name

OAuth 提供者返回的用户名称。

### nickName

用户公开可见的昵称。如果 OAuth 提供者没有昵称概念，`nickName` 和 `name` 的值将相同。

### avatarUrl

用户公开个人资料图片的 HTTP(s) URL。

### token

token 属性是对底层访问令牌对象的引用。token 对象具有以下子属性。

```ts
user.token.token
user.token.type
user.token.refreshToken
user.token.expiresAt
user.token.expiresIn
```

| 属性           | 协议            | 描述                                                                                      |
| -------------- | --------------- | ----------------------------------------------------------------------------------------- |
| `token`        | OAuth2 / OAuth1 | 访问令牌的值。该值可用于 `OAuth2` 和 `OAuth1` 协议。                                      |
| `secret`       | OAuth1          | 仅适用于 `OAuth1` 协议的令牌密钥。目前，Twitter 是唯一使用 OAuth1 的官方驱动程序。        |
| `type`         | OAuth2          | 令牌类型。通常，它将是 [bearer token](https://oauth.net/2/bearer-tokens/)。               |
| `refreshToken` | OAuth2          | 您可以使用刷新令牌创建新的访问令牌。如果 OAuth 提供者不支持刷新令牌，该值将为 `undefined` |
| `expiresAt`    | OAuth2          | luxon DateTime 类的实例，表示访问令牌过期的绝对时间。                                     |
| `expiresIn`    | OAuth2          | 令牌过期的秒数值。这是一个静态值，不会随时间变化。                                        |

### original

对 OAuth 提供者原始响应的引用。如果规范化的用户属性集没有您需要的所有信息，您可能需要引用原始响应。

```ts
const user = await github.user()
console.log(user.original)
```

## 定义作用域

作用域指的是用户批准认证请求后您想要访问的数据。作用域的名称和您可以访问的数据因 OAuth 提供者而异；因此，您必须阅读他们的文档。

作用域可以在 `config/ally.ts` 文件中定义，或者在重定向用户时定义。

感谢 TypeScript，您将获得所有可用作用域的自动完成建议。

![](../digging_deeper/ally_autocomplete.png)

```ts
// title: config/ally.ts
github: {
  driver: 'github',
  clientId: env.get('GITHUB_CLIENT_ID')!,
  clientSecret: env.get('GITHUB_CLIENT_SECRET')!,
  callbackUrl: '',
  // highlight-start
  scopes: ['read:user', 'repo:invite'],
  // highlight-end
}
```

```ts
// title: 重定向期间
ally.use('github').redirect((request) => {
  // highlight-start
  request.scopes(['read:user', 'repo:invite'])
  // highlight-end
})
```

## 定义重定向查询参数

您可以自定义重定向请求的查询参数以及作用域。在以下示例中，我们定义了适用于 [Google 提供者](https://developers.google.com/identity/protocols/oauth2/web-server#httprest) 的 `prompt` 和 `access_type` 参数。

```ts
router.get('/google/redirect', async ({ ally }) => {
  return ally.use('google').redirect((request) => {
    // highlight-start
    request.param('access_type', 'offline').param('prompt', 'select_account')
    // highlight-end
  })
})
```

您可以使用请求上的 `.clearParam()` 方法清除任何现有参数。如果配置中定义了参数默认值，并且您需要为单独的自定义认证流程重新定义它们，这会很有帮助。

```ts
router.get('/google/redirect', async ({ ally }) => {
  return ally.use('google').redirect((request) => {
    // highlight-start
    request.clearParam('redirect_uri').param('redirect_uri', '')
    // highlight-end
  })
})
```

## 从访问令牌获取用户详细信息

有时，您可能想要从存储在数据库中或通过另一个 OAuth 流程提供的访问令牌获取用户详细信息。例如，您通过移动应用使用了原生 OAuth 流程并收到了访问令牌。

您可以使用 `.userFromToken()` 方法获取用户详细信息。

```ts
const user = await ally.use('github').userFromToken(accessToken)
```

您可以使用 `.userFromTokenAndSecret` 方法为 OAuth1 驱动程序获取用户详细信息。

```ts
const user = await ally.use('github').userFromTokenAndSecret(token, secret)
```

## 无状态认证

许多 OAuth 提供者[建议使用 CSRF 状态令牌](https://developers.google.com/identity/openid-connect/openid-connect?hl=en#createxsrftoken)来防止您的应用程序受到跨站请求伪造攻击。

Ally 创建一个 CSRF 令牌并将其保存在加密 cookie 中，该 cookie 在用户批准认证请求后进行验证。

但是，如果您由于某种原因无法使用 cookie，您可以启用无状态模式，在这种模式下不会进行状态验证，因此不会生成 CSRF cookie。

```ts
// title: 重定向
ally.use('github').stateless().redirect()
```

```ts
// title: 处理回调响应
const gh = ally.use('github').stateless()
await gh.user()
```

## 完整配置参考

以下是所有驱动程序的完整配置参考。您可以直接将以下对象复制粘贴到 `config/ally.ts` 文件中。

<div class="disclosure_wrapper">

:::disclosure{title="GitHub 配置"}

```ts
{
  github: services.github({
    clientId: '',
    clientSecret: '',
    callbackUrl: '',

    // GitHub 特定
    login: 'adonisjs',
    scopes: ['user', 'gist'],
    allowSignup: true,
  })
}
```

:::

:::disclosure{title="Google 配置"}

```ts
{
  google: services.google({
    clientId: '',
    clientSecret: '',
    callbackUrl: '',

    // Google 特定
    prompt: 'select_account',
    accessType: 'offline',
    hostedDomain: 'adonisjs.com',
    display: 'page',
    scopes: ['userinfo.email', 'calendar.events'],
  })
}
```

:::

:::disclosure{title="Twitter 配置"}

```ts
{
  twitter: services.twitter({
    clientId: '',
    clientSecret: '',
    callbackUrl: '',
  })
}
```

:::

:::disclosure{title="Discord 配置"}

```ts
{
  discord: services.discord({
    clientId: '',
    clientSecret: '',
    callbackUrl: '',

    // Discord 特定
    prompt: 'consent' | 'none',
    guildId: '',
    disableGuildSelect: false,
    permissions: 10,
    scopes: ['identify', 'email'],
  })
}
```

:::

:::disclosure{title="LinkedIn 配置（已弃用）"}

此配置已弃用，以符合更新的 [LinkedIn OAuth 要求](https://learn.microsoft.com/en-us/linkedin/consumer/integrations/self-serve/sign-in-with-linkedin)。

```ts
{
  linkedin: services.linkedin({
    clientId: '',
    clientSecret: '',
    callbackUrl: '',

    // LinkedIn 特定
    scopes: ['r_emailaddress', 'r_liteprofile'],
  })
}
```

:::

:::disclosure{title="LinkedIn Openid Connect 配置"}

```ts
{
  linkedin: services.linkedinOpenidConnect({
    clientId: '',
    clientSecret: '',
    callbackUrl: '',

    // LinkedIn 特定
    scopes: ['openid', 'profile', 'email'],
  })
}
```

:::

:::disclosure{title="Facebook 配置"}

```ts
{
  facebook: services.facebook({
    clientId: '',
    clientSecret: '',
    callbackUrl: '',

    // Facebook 特定
    scopes: ['email', 'user_photos'],
    userFields: ['first_name', 'picture', 'email'],
    display: '',
    authType: '',
  })
}
```

:::

:::disclosure{title="Spotify 配置"}

```ts
{
  spotify: services.spotify({
    clientId: '',
    clientSecret: '',
    callbackUrl: '',

    // Spotify 特定
    scopes: ['user-read-email', 'streaming'],
    showDialog: false,
  })
}
```

:::

</div>

## 创建自定义社交驱动程序

我们创建了一个[入门工具包](https://github.com/adonisjs-community/ally-driver-boilerplate)来在 npm 上实现和发布自定义社交驱动程序。请阅读入门工具包的 README 以获取进一步说明。
