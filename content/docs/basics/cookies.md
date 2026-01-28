---
summary: 学习如何在 AdonisJS 中读取、写入和清除 cookies。
---

# Cookies

在 HTTP 请求期间，请求 cookies 会被自动解析。你可以使用 [request](./request.md) 对象读取 cookies，使用 [response](./response.md) 对象设置/清除 cookies。

```ts
// title: 读取 cookies
import router from '@adonisjs/core/services/router'

router.get('cart', async ({ request }) => {
  // highlight-start
  const cartItems = request.cookie('cart_items', [])
  // highlight-end
  console.log(cartItems)
})
```

```ts
// title: 设置 cookies
import router from '@adonisjs/core/services/router'

router.post('cart', async ({ request, response }) => {
  const id = request.input('product_id')
  // highlight-start
  response.cookie('cart_items', [{ id }])
  // highlight-end
})
```

```ts
// title: 清除 cookies
import router from '@adonisjs/core/services/router'

router.delete('cart', async ({ request, response }) => {
  // highlight-start
  response.clearCookie('cart_items')
  // highlight-end
})
```

## 配置

设置 cookies 的默认配置在 `config/app.ts` 文件中定义。你可以根据应用程序需求自由调整这些选项。

```ts
http: {
  cookie: {
    domain: '',
    path: '/',
    maxAge: '2h',
    httpOnly: true,
    secure: true,
    sameSite: 'lax',
    /**
     * 实验性属性
     * https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie#partitioned
     */
    partitioned: false,
    priority: 'medium',
  }
}
```

这些选项会转换为 [set-cookie 属性](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie#attributes)。`maxAge` 属性接受基于字符串的时间表达式，其值将被转换为秒。

在设置 cookies 时可以覆盖相同的选项集。

```ts
response.cookie('key', value, {
  domain: '',
  path: '/',
  maxAge: '2h',
  httpOnly: true,
  secure: true,
  sameSite: 'lax',
})
```

## 支持的数据类型

cookie 值使用 `JSON.stringify` 序列化为字符串；因此，你可以使用以下 JavaScript 数据类型作为 cookie 值。

- string
- number
- bigInt
- boolean
- null
- object
- array

```ts
// 对象
response.cookie('user', {
  id: 1,
  fullName: 'virk',
})

// 数组
response.cookie('product_ids', [1, 2, 3, 4])

// 布尔值
response.cookie('is_logged_in', true)

// 数字
response.cookie('visits', 10)

// BigInt
response.cookie('visits', BigInt(10))

// 日期对象转换为 ISO 字符串
response.cookie('visits', new Date())
```

## 签名 cookies

使用 `response.cookie` 方法设置的 cookies 会被签名。签名 cookie 是防篡改的，意味着更改其内容将使签名无效，cookie 将被忽略。

cookies 使用 `config/app.ts` 文件中定义的 `appKey` 进行签名。

:::note

签名 cookies 仍然可以通过 Base64 解码读取。如果你希望 cookie 值不可读，可以使用加密 cookies。

:::

```ts
import router from '@adonisjs/core/services/router'

router.get('/', async ({ request, response }) => {
  // 设置签名 cookie
  response.cookie('user_id', 1)

  // 读取签名 cookie
  request.cookie('user_id')
})
```

## 加密 cookies

与签名 cookies 不同，加密 cookie 值无法解码为纯文本。因此，你可以使用加密 cookies 来存储包含敏感信息的值，这些信息不应该被任何人读取。

加密 cookies 使用 `response.encryptedCookie` 方法设置，使用 `request.encryptedCookie` 方法读取。在底层，这些方法使用 [加密模块](../security/encryption.md)。

```ts
import router from '@adonisjs/core/services/router'

router.get('/', async ({ request, response }) => {
  // 设置加密 cookie
  response.encryptedCookie('user_id', 1)

  // 读取加密 cookie
  request.encryptedCookie('user_id')
})
```

## 普通 cookies

普通 cookies 用于你希望前端代码通过 `document.cookie` 值访问 cookie 的情况。

默认情况下，我们会对原始值调用 `JSON.stringify` 并将其转换为 base64 编码的字符串。这样做是为了让你可以将 `对象` 和 `数组` 用作 cookie 值。但是，编码可以被关闭。

普通 cookies 使用 `response.plainCookie` 方法定义，可以使用 `request.plainCookie` 方法读取。

```ts
import router from '@adonisjs/core/services/router'

router.get('/', async ({ request, response }) => {
  // 设置普通 cookie
  response.plainCookie(
    'user',
    { id: 1 },
    {
      httpOnly: true,
    },
  )

  // 读取普通 cookie
  request.plainCookie('user')
})
```

要关闭编码，请在选项对象中设置 `encoding: false`。

```ts
response.plainCookie('token', tokenValue, {
  httpOnly: true,
  encode: false,
})

// 关闭编码读取普通 cookie
request.plainCookie('token', {
  encoded: false,
})
```

## 在测试期间设置 cookies

以下指南涵盖了在编写测试时使用 cookies 的情况。

- 使用 [Japa API 客户端](../testing/http_tests.md#readingwriting-cookies) 定义 cookies。
- 使用 [Japa 浏览器客户端](../testing/browser_tests.md#readingwriting-cookies) 定义 cookie。
