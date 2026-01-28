---
summary: Response类用于发送HTTP响应。它支持发送HTML片段、JSON对象、流以及更多内容。
---

# 响应

[response类](https://github.com/adonisjs/http-server/blob/main/src/response.ts)的实例用于响应HTTP请求。AdonisJS支持发送**HTML片段**、**JSON对象**、**流**以及更多内容。响应实例可以通过`ctx.response`属性访问。

## 发送响应

发送响应最简单的方法是从路由处理器返回一个值。

```ts
import router from '@adonisjs/core/services/router'

router.get('/', async () => {
  /** 普通字符串 */
  return 'This is the homepage.'

  /** HTML片段 */
  return '<p> This is the homepage </p>'

  /** JSON响应 */
  return { page: 'home' }

  /** 转换为ISO字符串 */
  return new Date()
})
```

除了从路由处理器返回值外，你还可以使用`response.send`方法显式设置响应体。然而，多次调用`response.send`方法会覆盖旧的内容体，只保留最新的一个。

```ts
import router from '@adonisjs/core/services/router'

router.get('/', async ({ response }) => {
  /** 普通字符串 */
  response.send('This is the homepage')

  /** HTML片段 */
  response.send('<p> This is the homepage </p>')

  /** JSON响应 */
  response.send({ page: 'home' })

  /** 转换为ISO字符串 */
  response.send(new Date())
})
```

可以使用`response.status`方法为响应设置自定义状态码。

```ts
response.status(200).send({ page: 'home' })

// 发送空的201响应
response.status(201).send('')
```

## 流式传输内容

`response.stream`方法允许将流式数据传输到响应。该方法在完成后会在内部销毁流。

`response.stream`方法不会设置`content-type`和`content-length`头；你必须在流式传输内容之前显式设置它们。

```ts
import router from '@adonisjs/core/services/router'

router.get('/', async ({ response }) => {
  const image = fs.createReadStream('./some-file.jpg')
  response.stream(image)
})
```

如果发生错误，会向客户端发送500状态码。然而，你可以通过定义回调函数作为第二个参数来自定义错误代码和消息。

```ts
const image = fs.createReadStream('./some-file.jpg')

response.stream(image, () => {
  const message = 'Unable to serve file. Try again'
  const status = 400

  return [message, status]
})
```

## 下载文件

当你想要从磁盘流式传输文件时，我们推荐使用`response.download`方法而不是`response.stream`方法。这是因为`download`方法会自动设置`content-type`和`content-length`头。

```ts
import app from '@adonisjs/core/services/app'
import router from '@adonisjs/core/services/router'

router.get('/uploads/:file', async ({ response, params }) => {
  const filePath = app.makePath(`uploads/${params.file}`)

  response.download(filePath)
})
```

可选择性地，你可以为文件内容生成一个[Etag](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/ETag)。使用Etags将帮助浏览器重用之前请求的缓存响应（如果有的话）。

```ts
const filePath = app.makePath(`uploads/${params.file}`)
const generateEtag = true

response.download(filePath, generateEtag)
```

与`response.stream`方法类似，你可以通过定义回调函数作为最后一个参数来发送自定义错误消息和状态码。

```ts
const filePath = app.makePath(`uploads/${params.file}`)
const generateEtag = true

response.download(filePath, generateEtag, (error) => {
  if (error.code === 'ENOENT') {
    return ['File does not exists', 404]
  }

  return ['Cannot download file', 400]
})
```

### 强制下载文件

`response.attachment`方法与`response.download`方法类似，但它通过设置[Content-Disposition](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Disposition)头来强制浏览器将文件保存到用户的计算机上。

```ts
import app from '@adonisjs/core/services/app'
import router from '@adonisjs/core/services/router'

router.get('/uploads/:file', async ({ response, params }) => {
  const filePath = app.makePath(`uploads/${params.file}`)

  response.attachment(filePath, 'custom-filename.jpg')
})
```

## 设置响应状态码和头部

### 设置状态码

你可以使用`response.status`方法设置响应状态码。调用此方法会覆盖现有的响应状态码（如果有的话）。然而，你可以使用`response.safeStatus`方法仅在状态码为`undefined`时设置它。

```ts
import router from '@adonisjs/core/services/router'

router.get('/', async ({ response }) => {
  /**
   * 将状态码设置为200
   */
  response.safeStatus(200)

  /**
   * 不设置状态码，因为它
   * 已经被设置了
   */
  response.safeStatus(201)
})
```

### 设置头部

你可以使用`response.header`方法设置响应头部。此方法会覆盖现有的头部值（如果已存在）。然而，你可以使用`response.safeHeader`方法仅在头部为`undefined`时设置它。

```ts
import router from '@adonisjs/core/services/router'

router.get('/', async ({ response }) => {
  /**
   * 定义content-type头部
   */
  response.safeHeader('Content-type', 'text/html')

  /**
   * 不设置content-type头部，因为它
   * 已经被设置了
   */
  response.safeHeader('Content-type', 'text/html')
})
```

你可以使用`response.append`方法向现有头部值追加值。

```ts
response.append('Set-cookie', 'cookie-value')
```

`response.removeHeader`方法会移除现有的头部。

```ts
response.removeHeader('Set-cookie')
```

### X-Request-Id头部

如果当前请求中存在该头部，或者启用了[生成请求ID](./request#generating-request-ids)，该头部将会出现在响应中。

## 重定向

`response.redirect`方法返回[Redirect](https://github.com/adonisjs/http-server/blob/main/src/redirect.ts)类的实例。重定向类使用流畅的API来构建重定向URL。

执行重定向最简单的方法是使用重定向路径调用`redirect.toPath`方法。

```ts
import router from '@adonisjs/core/services/router'

router.get('/posts', async ({ response }) => {
  response.redirect().toPath('/articles')
})
```

重定向类还允许从预注册的路由构建URL。`redirect.toRoute`方法接受[路由标识符](./routing.md#route-identifier)作为第一个参数，路由参数作为第二个参数。

```ts
import router from '@adonisjs/core/services/router'

router.get('/articles/:id', async () => {}).as('articles.show')

router.get('/posts/:id', async ({ response, params }) => {
  response.redirect().toRoute('articles.show', { id: params.id })
})
```

### 重定向回上一页

你可能想要在表单提交期间出现验证错误时将用户重定向回上一页。你可以使用`redirect.back`方法来实现这一点。

```ts
response.redirect().back()
```

### 重定向状态码

重定向响应的默认状态码是`302`；你可以通过调用`redirect.status`方法来改变它。

```ts
response.redirect().status(301).toRoute('articles.show', { id: params.id })
```

### 带查询字符串的重定向

你可以使用`withQs`方法将查询字符串附加到重定向URL。该方法接受一个键值对对象并将其转换为字符串。

```ts
response.redirect().withQs({ page: 1, limit: 20 }).toRoute('articles.index')
```

要从当前请求URL转发查询字符串，调用不带任何参数的`withQs`方法。

```ts
// 转发当前URL查询字符串
response.redirect().withQs().toRoute('articles.index')
```

当重定向回上一页时，`withQs`方法会转发上一页的查询字符串。

```ts
// 转发当前URL查询字符串
response.redirect().withQs().back()
```

## 使用错误中止请求

你可以使用`response.abort`方法通过引发异常来结束请求。该方法会抛出`E_HTTP_REQUEST_ABORTED`异常并触发[异常处理](./exception_handling.md)流程。

```ts
router.get('posts/:id/edit', async ({ response, auth, params }) => {
  const post = await Post.findByOrFail(params.id)

  if (!auth.user.can('editPost', post)) {
    response.abort({ message: 'Cannot edit post' })
  }

  // 继续执行其余逻辑
})
```

默认情况下，异常会创建一个带有`400`状态码的HTTP响应。然而，你可以指定自定义状态码作为第二个参数。

```ts
response.abort({ message: 'Cannot edit post' }, 403)
```

## 在响应完成后执行操作

你可以使用`response.onFinish`方法监听Node.js完成将响应写入TCP套接字的事件。在底层，我们使用[on-finished](https://github.com/jshttp/on-finished)包，因此请随时查阅该包的README文件以获取深入的技术解释。

```ts
router.get('posts', ({ response }) => {
  response.onFinish(() => {
    // 清理逻辑
  })
})
```

## 访问Node.js的`res`对象

你可以使用`response.response`属性访问[Node.js的res对象](https://nodejs.org/dist/latest-v19.x/docs/api/http.html#class-httpserverresponse)。

```ts
router.get('posts', ({ response }) => {
  console.log(response.response)
})
```

## 响应体序列化

使用`response.send`方法设置的响应体在被[写入响应](https://nodejs.org/dist/latest-v18.x/docs/api/http.html#responsewritechunk-encoding-callback)到传出消息流之前会被序列化为字符串。

以下是支持的数据类型及其序列化规则：

- 数组和对象使用[安全字符串化函数](https://github.com/poppinss/utils/blob/main/src/json/safe_stringify.ts)进行字符串化。该方法类似于`JSON.stringify`，但会移除循环引用并序列化`BigInt(s)`。
- 数字和布尔值会被转换为字符串。
- Date类的实例会通过调用`toISOString`方法转换为字符串。
- 正则表达式和错误对象会通过调用`toString`方法转换为字符串。
- 任何其他数据类型都会导致异常。

### 内容类型推断

在序列化响应后，响应类会自动推断并设置`content-type`和`content-length`头部。

以下是我们设置`content-type`头部所遵循的规则列表：

- 数组和对象的内容类型设置为`application/json`。
- HTML片段的内容类型设置为`text/html`。
- JSONP响应以`text/javascript`内容类型发送。
- 其他所有内容的内容类型设置为`text/plain`。

## 扩展Response类

你可以使用宏或getter向Response类添加自定义属性。如果你不熟悉宏的概念，请先阅读[扩展AdonisJS指南](../concepts/extending_the_framework.md)。

```ts
import { Response } from '@adonisjs/core/http'

Response.macro('property', function (this: Response) {
  return value
})
Response.getter('property', function (this: Response) {
  return value
})
```

由于宏和getter是在运行时添加的，你必须告知TypeScript它们的类型。

```ts
declare module '@adonisjs/core/http' {
  export interface Response {
    property: valueType
  }
}
```
