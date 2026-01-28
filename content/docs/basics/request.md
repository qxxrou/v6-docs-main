---
summary: Request 类保存了当前 HTTP 请求的所有数据，包括请求体、上传文件引用、Cookie、请求头等信息。
---

# 请求

[request 类](https://github.com/adonisjs/http-server/blob/main/src/request.ts) 的实例保存了当前 HTTP 请求的所有数据，包括**请求体**、**上传文件引用**、**Cookie**、**请求头**等。可以通过 `ctx.request` 属性访问该请求实例。

## 查询字符串与路由参数

`request.qs` 方法将查询字符串解析为对象并返回。

```ts
import router from '@adonisjs/core/services/router'

router.get('posts', async ({ request }) => {
  /*
   * URL: /?sort_by=id&direction=desc
   * qs: { sort_by: 'id', direction: 'desc' }
   */
  request.qs()
})
```

`request.params` 方法返回 [路由参数](./routing.md#route-params) 对象。

```ts
import router from '@adonisjs/core/services/router'

router.get('posts/:slug/comments/:id', async ({ request }) => {
  /*
   * URL: /posts/hello-world/comments/2
   * params: { slug: 'hello-world', id: '2' }
   */
  request.params()
})
```

使用 `request.param` 方法可获取单个参数值。

```ts
import router from '@adonisjs/core/services/router'

router.get('posts/:slug/comments/:id', async ({ request }) => {
  const slug = request.param('slug')
  const commentId = request.param('id')
})
```

## 请求体

AdonisJS 通过注册在 `start/kernel.ts` 中的 [body-parser 中间件](../basics/body_parser.md) 解析请求体。

使用 `request.body()` 方法即可拿到解析后的请求体对象。

```ts
import router from '@adonisjs/core/services/router'

router.post('/', async ({ request }) => {
  console.log(request.body())
})
```

`request.all` 会返回请求体与查询字符串合并后的副本。

```ts
import router from '@adonisjs/core/services/router'

router.post('/', async ({ request }) => {
  console.log(request.all())
})
```

### 按需取值

`request.input`、`request.only`、`request.except` 可从请求数据中按需提取字段，它们会同时在请求体和查询字符串里查找。

`request.only` 仅返回指定字段组成的对象。

```ts
import router from '@adonisjs/core/services/router'

router.post('login', async ({ request }) => {
  const credentials = request.only(['email', 'password'])

  console.log(credentials)
})
```

`request.except` 返回排除指定字段后的对象。

```ts
import router from '@adonisjs/core/services/router'

router.post('register', async ({ request }) => {
  const userDetails = request.except(['password_confirmation'])

  console.log(userDetails)
})
```

`request.input` 获取单个字段值，可传入默认值作为第二参数，当字段缺失时返回默认值。

```ts
import router from '@adonisjs/core/services/router'

router.post('comments', async ({ request }) => {
  const email = request.input('email')
  const commentBody = request.input('body')
})
```

### 类型安全的请求体

默认情况下，AdonisJS 不会对 `request.all`、`request.body` 及按需取值方法做类型校验，因为它无法预知请求体内容。

如需类型安全，请使用 [验证器（validator）](./validation.md) 对请求体进行校验与解析，确保字段类型符合预期。

## 请求 URL

`request.url` 返回相对于主机的请求路径，默认不含查询字符串；传入 `true` 即可包含查询字符串。

```ts
import router from '@adonisjs/core/services/router'

router.get('/users', async ({ request }) => {
  /*
   * URL: /users?page=1&limit=20
   * url: /users
   */
  request.url()

  /*
   * URL: /users?page=1&limit=20
   * url: /users?page=1&limit=20
   */
  request.url(true) // 返回带查询字符串的 URL
})
```

`request.completeUrl` 返回包含主机的完整 URL，同样默认不带查询字符串，需要时可传入 `true`。

```ts
import router from '@adonisjs/core/services/router'

router.get('/users', async ({ request }) => {
  request.completeUrl()
  request.completeUrl(true) // 返回带查询字符串的完整 URL
})
```

## 请求头

`request.headers` 返回所有请求头组成的对象。

```ts
import router from '@adonisjs/core/services/router'

router.get('/', async ({ request }) => {
  console.log(request.headers())
})
```

使用 `request.header` 获取单个头信息，字段名不区分大小写。

```ts
import router from '@adonisjs/core/services/router'

router.get('/', async ({ request }) => {
  request.header('x-request-id')

  // 字段名不区分大小写
  request.header('X-REQUEST-ID')
})
```

## 请求方法

`request.method` 返回当前匹配的 HTTP 方法。若启用了 [表单方法伪装](#form-method-spoofing)，则返回伪装后的方法；使用 `request.intended` 可获取原始方法。

```ts
import router from '@adonisjs/core/services/router'

router.patch('posts', async ({ request }) => {
  /**
   * 用于路由匹配的方法
   */
  console.log(request.method())

  /**
   * 实际请求方法
   */
  console.log(request.intended())
})
```

## 用户 IP 地址

`request.ip` 返回当前请求的用户 IP，依赖 Nginx、Caddy 等代理设置的 [`X-Forwarded-For`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Forwarded-For) 头。

:::note

阅读 [受信代理](#configuring-trusted-proxies) 章节，配置应用应信任的代理。

:::

```ts
import router from '@adonisjs/core/services/router'

router.get('/', async ({ request }) => {
  console.log(request.ip())
})
```

`request.ips` 返回中间代理设置的所有 IP 地址数组，按信任度从高到低排序。

```ts
import router from '@adonisjs/core/services/router'

router.get('/', async ({ request }) => {
  console.log(request.ips())
})
```

### 自定义 `getIp` 方法

若受信代理配置仍无法满足需求，可在 `config/app.ts` 的 `http` 配置中实现自定义 `getIp` 方法。

```ts
export const http = defineConfig({
  getIp(request) {
    const ip = request.header('X-Real-Ip')
    if (ip) {
      return ip
    }

    return request.ips()[0]
  },
})
```

## 内容协商

AdonisJS 提供一系列方法，通过解析常见的 `Accept` 头实现 [内容协商](https://developer.mozilla.org/en-US/docs/Web/HTTP/Content_negotiation#server-driven_content_negotiation)。例如，用 `request.types` 获取请求支持的内容类型列表，返回值按客户端偏好排序。

```ts
import router from '@adonisjs/core/services/router'

router.get('/', async ({ request }) => {
  console.log(request.types())
})
```

内容协商方法完整列表：

| 方法      | 对应 HTTP 头                                                                                 |
| --------- | -------------------------------------------------------------------------------------------- |
| types     | [Accept](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept)                   |
| languages | [Accept-Language](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept-Language) |
| encodings | [Accept-Encoding](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept-Encoding) |
| charsets  | [Accept-Charset](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept-Charset)   |

若需根据服务器支持的内容类型选择最佳匹配，可使用 `request.accepts`。传入服务器支持的类型数组，方法会依据 `Accept` 头返回最优先匹配项，无匹配时返回 `null`。

```ts
import router from '@adonisjs/core/services/router'

router.get('posts', async ({ request, view }) => {
  const posts = [
    {
      title: 'Adonis 101',
    },
  ]

  const bestMatch = request.accepts(['html', 'json'])

  switch (bestMatch) {
    case 'html':
      return view.render('posts/index', { posts })
    case 'json':
      return posts
    default:
      return view.render('posts/index', { posts })
  }
})
```

类似地，以下方法用于其他 `Accept` 头的最优值选择：

```ts
// 最优语言
const language = request.language(['fr', 'de'])

// 最优编码
const encoding = request.encoding(['gzip', 'br'])

// 最优字符集
const charset = request.charset(['utf-8', 'hex', 'ascii'])
```

## 生成请求 ID

请求 ID 能为每次 HTTP 请求分配唯一标识，方便在日志中 [追踪与调试](https://blog.heroku.com/http_request_id_s_improve_visibility_across_the_application_stack)。默认关闭，可在 `config/app.ts` 中开启。

:::note

请求 ID 由 [cuid2](https://github.com/paralleldrive/cuid2) 生成。生成前会检查 `X-Request-Id` 请求头，若存在则直接使用。

:::

```ts
// title: config/app.ts
export const http = defineConfig({
  generateRequestId: true,
})
```

开启后，通过 `request.id` 获取当前请求 ID。

```ts
router.get('/', ({ request }) => {
  // ckk9oliws0000qt3x9vr5dkx7
  console.log(request.id())
})
```

所有使用 `ctx.logger` 记录的日志都会自动带上该请求 ID。

```ts
router.get('/', ({ logger }) => {
  // { msg: 'hello world', request_id: 'ckk9oliws0000qt3x9vr5dkx7' }
  logger.info('hello world')
})
```

## 配置受信代理

多数 Node.js 应用部署在 Nginx、Caddy 等代理之后，需要依赖 `X-Forwarded-Host`、`X-Forwarded-For`、`X-Forwarded-Proto` 等头获取真实客户端信息。

仅当应用信任来源 IP 时，才会使用这些头。可在 `config/app.ts` 通过 `http.trustProxy` 配置受信 IP。

```ts
import proxyAddr from 'proxy-addr'

export const http = defineConfig({
  trustProxy: proxyAddr.compile(['127.0.0.1/8', '::1/128']),
})
```

`trustProxy` 也支持函数形式，返回 `true` 表示信任该地址。

```ts
export const http = defineConfig({
  trustProxy: (address) => {
    return address === '127.0.0.1' || address === '123.123.123.123'
  },
})
```

若 Nginx 与应用同机，信任回环地址即可：

```ts
import proxyAddr from 'proxy-addr'

export const http = defineConfig({
  trustProxy: proxyAddr.compile('loopback'),
})
```

若应用仅通过负载均衡访问，且无固定 IP 列表，可始终返回 `true` 信任代理：

```ts
export const http = defineConfig({
  trustProxy: () => true,
})
```

## 配置查询字符串解析器

查询字符串由 [qs](http://npmjs.com/qs) 模块解析，可在 `config/app.ts` 中调整解析选项。

查看 [全部可用选项](https://github.com/adonisjs/http-server/blob/main/src/types/qs.ts#L11)。

```ts
export const http = defineConfig({
  qs: {
    parse: {},
  },
})
```

## 表单方法伪装

HTML 表单仅支持 `GET` 或 `POST`，无法直接使用 [RESTful 方法](https://restfulapi.net/http-methods/)。AdonisJS 支持通过 **\_method** 查询参数进行 **表单方法伪装**。

启用需在表单设置 `method="POST"` 并在 `config/app.ts` 打开开关：

```ts
// title: config/app.ts
export const http = defineConfig({
  allowMethodSpoofing: true,
})
```

启用后，即可如下伪装：

```html
<form method="POST" action="/articles/1?_method=PUT">
  <!-- 更新表单 -->
</form>
```

```html
<form method="POST" action="/articles/1?_method=DELETE">
  <!-- 删除表单 -->
</form>
```

## 扩展 Request 类

可通过宏（macro）或 getter 为 Request 类添加自定义属性。首次接触请先阅读 [扩展 AdonisJS 指南](../concepts/extending_the_framework.md)。

```ts
import { Request } from '@adonisjs/core/http'

Request.macro('property', function (this: Request) {
  return value
})
Request.getter('property', function (this: Request) {
  return value
})
```

由于宏与 getter 在运行时添加，需要向 TypeScript 声明类型：

```ts
declare module '@adonisjs/core/http' {
  export interface Request {
    property: valueType
  }
}
```
