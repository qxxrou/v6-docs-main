---
summary: 在 AdonisJS 中使用 Japa API 客户端插件进行 HTTP 测试。
---

# HTTP 测试

HTTP 测试指的是通过对你的应用程序端点进行实际 HTTP 请求并断言响应主体、标头、cookie、会话等来测试你的应用程序端点。

HTTP 测试使用 Japa 的 [API 客户端插件](https://japa.dev/docs/plugins/api-client) 执行。API 客户端插件是一个无状态请求库，类似于 `Axios` 或 `fetch`，但更适合测试。

如果你想在真实浏览器中测试你的 Web 应用程序并以编程方式与它们交互，我们建议使用使用 Playwright 进行测试的 [浏览器客户端](./browser_tests.md)。

## 设置

第一步是从 npm 包注册表安装以下包。

:::codegroup

```sh
// title: npm
npm i -D @japa/api-client
```

:::

### 注册插件

在继续之前，请在 `tests/bootstrap.ts` 文件中注册插件。

```ts
// title: tests/bootstrap.ts
import { apiClient } from '@japa/api-client'

export const plugins: Config['plugins'] = [
  assert(),
  // highlight-start
  apiClient(),
  // highlight-end
  pluginAdonisJS(app),
]
```

`apiClient` 方法可选择接受服务器的 `baseURL`。如果未提供，它将使用 `HOST` 和 `PORT` 环境变量。

```ts
import env from '#start/env'

export const plugins: Config['plugins'] = [
  apiClient({
    baseURL: `http://${env.get('HOST')}:${env.get('PORT')}`,
  }),
]
```

## 基本示例

注册 `apiClient` 插件后，你可以从 [TestContext](https://japa.dev/docs/test-context) 访问 `client` 对象以发出 HTTP 请求。

HTTP 测试必须在为 `functional` 测试套件配置的文件夹内编写。你可以使用以下命令创建新的测试文件。

```sh
node ace make:test users/list --suite=functional
```

```ts
import { test } from '@japa/runner'

test.group('用户列表', () => {
  test('获取用户列表', async ({ client }) => {
    const response = await client.get('/users')

    response.assertStatus(200)
    response.assertBody({
      data: [
        {
          id: 1,
          email: 'foo@bar.com',
        },
      ],
    })
  })
})
```

要查看所有可用的请求和断言方法，请务必[查看 Japa 文档](https://japa.dev/docs/plugins/api-client)。

## Open API 测试

断言和 API 客户端插件允许你使用 Open API 规范文件编写断言。你可以使用规范文件测试 HTTP 响应的形状，而不是手动测试响应与固定负载。

这是保持 Open API 规范和服务器响应同步的好方法。因为如果你从规范文件中删除某个端点或更改响应数据形状，你的测试将失败。

### 注册模式

AdonisJS 不提供用于从代码生成 Open API 模式文件的工具。你可以手动编写它或使用图形工具创建它。

获得规范文件后，将其保存在 `resources` 目录中（如果缺少则创建目录），并在 `tests/bootstrap.ts` 文件中注册 `openapi-assertions` 插件。

```sh
npm i -D @japa/openapi-assertions
```

```ts
// title: tests/bootstrap.ts
import app from '@adonisjs/core/services/app'
// highlight-start
import { openapi } from '@japa/openapi-assertions'
// highlight-end

export const plugins: Config['plugins'] = [
  assert(),
  // highlight-start
  openapi({
    schemas: [app.makePath('resources/open_api_schema.yaml')]
  })
  // highlight-end
  apiClient(),
  pluginAdonisJS(app)
]
```

### 编写断言

注册模式后，你可以使用 `response.assertAgainstApiSpec` 方法对 API 规范进行断言。

```ts
test('分页帖子', async ({ client }) => {
  const response = await client.get('/posts')
  response.assertAgainstApiSpec()
})
```

- `response.assertAgainstApiSpec` 方法将使用 **请求方法**、**端点** 和 **响应状态代码** 来查找预期的响应模式。
- 当找不到响应模式时，将引发异常。否则，响应主体将根据模式进行验证。

仅测试响应的形状，而不是实际值。因此，你可能必须编写额外的断言。例如：

```ts
// 断言响应符合模式
response.assertAgainstApiSpec()

// 断言预期值
response.assertBodyContains({
  data: [{ title: 'Adonis 101' }, { title: 'Lucid 101' }],
})
```

## 读取/写入 cookie

你可以使用 `withCookie` 方法在 HTTP 请求期间发送 cookie。该方法接受 cookie 名称作为第一个参数，值作为第二个参数。

```ts
await client.get('/users').withCookie('user_preferences', { limit: 10 })
```

`withCookie` 方法定义了一个[签名 cookie](../basics/cookies.md#signed-cookies)。此外，你可以使用 `withEncryptedCookie` 或 `withPlainCookie` 方法在请求期间发送其他类型的 cookie 到服务器。

```ts
await client
  .get('/users')
  .withEncryptedCookie('user_preferences', { limit: 10 })
```

```ts
await client.get('/users').withPlainCookie('user_preferences', { limit: 10 })
```

### 从响应中读取 cookie

你可以使用 `response.cookies` 方法访问由 AdonisJS 服务器设置的 cookie。该方法返回 cookie 作为键值对的对象。

```ts
const response = await client.get('/users')
console.log(response.cookies())
```

你可以使用 `response.cookie` 方法按名称访问单个 cookie 值。或使用 `assertCookie` 方法断言 cookie 值。

```ts
const response = await client.get('/users')

console.log(response.cookie('user_preferences'))

response.assertCookie('user_preferences')
```

## 填充会话存储

如果你在应用程序中使用 [`@adonisjs/session`](../basics/session.md) 包来读取/写入会话数据，你可能还希望在编写测试时使用 `sessionApiClient` 插件来填充会话存储。

### 设置

第一步是在 `tests/bootstrap.ts` 文件中注册插件。

```ts
// title: tests/bootstrap.ts
// insert-start
import { sessionApiClient } from '@adonisjs/session/plugins/api_client'
// insert-end

export const plugins: Config['plugins'] = [
  assert(),
  pluginAdonisJS(app),
  // insert-start
  sessionApiClient(app),
  // insert-end
]
```

然后，更新 `.env.test` 文件（如果缺少则创建一个）并将 `SESSON_DRIVER` 设置为 `memory`。

```dotenv
// title: .env.test
SESSION_DRIVER=memory
```

### 使用会话数据发出请求

你可以使用 Japa API 客户端上的 `withSession` 方法发出带有某些预定义会话数据的 HTTP 请求。

`withSession` 方法将创建一个新的会话 ID 并用数据填充内存存储，你的 AdonisJS 应用程序代码可以像往常一样读取会话数据。

请求完成后，会话 ID 及其数据将被销毁。

```ts
test('使用购物车项目结账', async ({ client }) => {
  await client
    .post('/checkout')
    // highlight-start
    .withSession({
      cartItems: [
        {
          id: 1,
          name: '南印度滤压咖啡',
        },
        {
          id: 2,
          name: '冷萃袋',
        },
      ],
    })
  // highlight-end
})
```

与 `withSession` 方法类似，你可以使用 `withFlashMessages` 方法在发出 HTTP 请求时设置闪存消息。

```ts
const response = await client.get('posts/1').withFlashMessages({
  success: '帖子创建成功',
})

response.assertTextIncludes(`帖子创建成功`)
```

### 从响应中读取会话数据

你可以使用 `response.session()` 方法访问由 AdonisJS 服务器设置的会话数据。该方法返回会话数据作为键值对的对象。

```ts
const response = await client.post('/posts').json({
  title: '某些标题',
  body: '某些描述',
})

console.log(response.session()) // 所有会话数据
console.log(response.session('post_id')) // post_id 的值
```

你可以使用 `response.flashMessage` 或 `response.flashMessages` 方法从响应中读取闪存消息。

```ts
const response = await client.post('/posts')

response.flashMessages()
response.flashMessage('errors')
response.flashMessage('success')
```

最后，你可以使用以下方法之一对会话数据编写断言。

```ts
const response = await client.post('/posts')

/**
 * 断言会话存储中存在特定键（带有可选值）
 */
response.assertSession('cart_items')
response.assertSession('cart_items', [
  {
    id: 1,
  },
  {
    id: 2,
  },
])

/**
 * 断言会话存储中不存在特定键
 */
response.assertSessionMissing('cart_items')

/**
 * 断言闪存消息存储中存在闪存消息（带有可选值）
 */
response.assertFlashMessage('success')
response.assertFlashMessage('success', '帖子创建成功')

/**
 * 断言闪存消息存储中不存在特定键
 */
response.assertFlashMissing('errors')

/**
 * 断言闪存消息中的验证错误
 * 存储
 */
response.assertHasValidationError('title')
response.assertValidationError('title', '输入帖子标题')
response.assertValidationErrors('title', [
  '输入帖子标题',
  '帖子标题必须为 10 个字符长。',
])
response.assertDoesNotHaveValidationError('title')
```

## 验证用户

如果你在应用程序中使用 `@adonisjs/auth` 包来验证用户，你可以使用 `authApiClient` Japa 插件在向应用程序发出 HTTP 请求时验证用户。

第一步是在 `tests/bootstrap.ts` 文件中注册插件。

```ts
// title: tests/bootstrap.ts
// insert-start
import { authApiClient } from '@adonisjs/auth/plugins/api_client'
// insert-end

export const plugins: Config['plugins'] = [
  assert(),
  pluginAdonisJS(app),
  // insert-start
  authApiClient(app),
  // insert-end
]
```

如果你使用基于会话的身份验证，请确保也设置会话插件。请参阅[填充会话存储 - 设置](#setup-1)。

就这样。现在，你可以使用 `loginAs` 方法登录用户。该方法接受用户对象作为唯一参数，并将用户标记为当前 HTTP 请求的登录用户。

```ts
import User from '#models/user'

test('获取付款列表', async ({ client }) => {
  const user = await User.create(payload)

  await client
    .get('/me/payments')
    // highlight-start
    .loginAs(user)
  // highlight-end
})
```

`loginAs` 方法使用 `config/auth.ts` 文件中配置的默认保护进行身份验证。但是，你可以使用 `withGuard` 方法指定自定义保护。例如：

```ts
await client
  .get('/me/payments')
  // highlight-start
  .withGuard('api_tokens')
  .loginAs(user)
// highlight-end
```

## 使用 CSRF 令牌发出请求

如果你应用程序中的表单使用 [CSRF 保护](../security/securing_ssr_applications.md)，你可以使用 `withCsrfToken` 方法生成 CSRF 令牌并在请求期间将其作为标头传递。

在使用 `withCsrfToken` 方法之前，请在 `tests/bootstrap.ts` 文件中注册以下 Japa 插件，并确保[将 `SESSION_DRIVER` 环境变量切换到](#setup-1) `memory`。

```ts
// title: tests/bootstrap.ts
// insert-start
import { shieldApiClient } from '@adonisjs/shield/plugins/api_client'
import { sessionApiClient } from '@adonisjs/session/plugins/api_client'
// insert-end

export const plugins: Config['plugins'] = [
  assert(),
  pluginAdonisJS(app),
  // insert-start
  sessionApiClient(app),
  shieldApiClient(),
  // insert-end
]
```

```ts
test('创建帖子', async ({ client }) => {
  await client.post('/posts').form(dataGoesHere).withCsrfToken()
})
```

## 路由助手

你可以从 TestContext 使用 `route` 助手来创建路由的 URL。使用路由助手可确保每当你更新路由时，不必返回并修复测试内的所有 URL。

`route` 助手接受全局模板方法 [route](../basics/routing.md#url-builder) 接受的同一组参数。

```ts
test('获取用户列表', async ({ client, route }) => {
  const response = await client.get(
    // highlight-start
    route('users.list'),
    // highlight-end
  )

  response.assertStatus(200)
  response.assertBody({
    data: [
      {
        id: 1,
        email: 'foo@bar.com',
      },
    ],
  })
})
```
