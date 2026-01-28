---
summary: 学习如何在 AdonisJS 中定义路由、路由参数和路由处理程序。
---

# 路由

您的网站或 Web 应用程序的用户可以访问不同的 URL，如 `/`、`/about` 或 `/posts/1`。为了让这些 URL 正常工作，您必须定义路由。

在 AdonisJS 中，路由定义在 `start/routes.ts` 文件中。路由是 **URI 模式** 和 **处理程序** 的组合，用于处理该特定路由的请求。例如：

```ts
// title: start/routes.ts
import router from '@adonisjs/core/services/router'

router.get('/', () => {
  return 'Hello world from the home page.'
})

router.get('/about', () => {
  return 'This is the about page.'
})

router.get('/posts/:id', ({ params }) => {
  return `This is post with id ${params.id}`
})
```

上面示例中的最后一个路由使用了动态 URI 模式。`id` 是一种告诉路由器接受任何值作为 id 的方式。我们称之为 **路由参数**。

## 查看已注册路由列表

您可以运行 `list:routes` 命令来查看应用程序注册的路由列表。

```sh
node ace list:routes
```

此外，如果您使用我们的 [官方 VSCode 扩展](https://marketplace.visualstudio.com/items?itemName=jripouteau.adonis-vscode-extension)，还可以从 VSCode 活动栏查看路由列表。

![](./vscode_routes_list.png)

## 路由参数

路由参数允许您定义可以接受动态值的 URI。每个参数都会捕获 URI 段的值，您可以在路由处理程序中访问此值。

路由参数始终以冒号 `:` 开头，后跟参数名称。.

```ts
// title: start/routes.ts
import router from '@adonisjs/core/services/router'

router.get('/posts/:id', ({ params }) => {
  return params.id
})
```

| URL              | Id        |
| ---------------- | --------- |
| `/posts/1`       | `1`       |
| `/posts/100`     | `100`     |
| `/posts/foo-bar` | `foo-bar` |

URI 也可以接受多个参数。每个参数都应该有唯一的名称。

```ts
// title: start/routes.ts
import router from '@adonisjs/core/services/router'

router.get('/posts/:id/comments/:commentId', ({ params }) => {
  console.log(params.id)
  console.log(params.commentId)
})
```

| URL                          | Id        | Comment Id |
| ---------------------------- | --------- | ---------- |
| `/posts/1/comments/4`        | `1`       | `4`        |
| `/posts/foo-bar/comments/22` | `foo-bar` | `22`       |

### 可选参数

路由参数也可以是可选的，只需在参数名称末尾添加问号 `?`。可选参数应放在必需参数之后。

```ts
// title: start/routes.ts
import router from '@adonisjs/core/services/router'

router.get('/posts/:id?', ({ params }) => {
  if (!params.id) {
    return 'Showing all posts'
  }

  return `Showing post with id ${params.id}`
})
```

### 通配符参数

要捕获 URI 的所有段，您可以定义通配符参数。通配符参数使用特殊的 `*` 关键字指定，并且必须定义在最后一个位置。

```ts
// title: start/routes.ts
import router from '@adonisjs/core/services/router'

router.get('/docs/:category/*', ({ params }) => {
  console.log(params.category)
  console.log(params['*'])
})
```

| URL                  | Category | Wildcard param   |
| -------------------- | -------- | ---------------- |
| `/docs/http/context` | `http`   | `['context']`    |
| `/docs/api/sql/orm`  | `api`    | `['sql', 'orm']` |

### 参数匹配器

路由器不知道您想要接受的参数数据格式。例如，URI 为 `/posts/foo-bar` 和 `/posts/1` 的请求将匹配相同的路由。但是，您可以使用参数匹配器显式验证参数值。

通过链式调用 `where()` 方法来注册匹配器。第一个参数是参数名称，第二个参数是匹配器对象。

在以下示例中，我们定义了一个正则表达式来验证 id 是否为有效数字。如果验证失败，将跳过该路由。

```ts
// title: start/routes.ts
import router from '@adonisjs/core/services/router'

router
  .get('/posts/:id', ({ params }) => {})
  .where('id', {
    match: /^[0-9]+$/,
  })
```

除了 `match` 正则表达式外，您还可以定义 `cast` 函数将参数值转换为其正确的数据类型。在这个例子中，我们可以将 id 转换为数字。

```ts
// title: start/routes.ts
import router from '@adonisjs/core/services/router'

router
  .get('/posts/:id', ({ params }) => {
    console.log(typeof params.id)
  })
  .where('id', {
    match: /^[0-9]+$/,
    cast: (value) => Number(value),
  })
```

### 内置匹配器

路由器为常用的数据类型提供了以下辅助方法。

```ts
// title: start/routes.ts
import router from '@adonisjs/core/services/router'

// Validate id to be numeric + cast to number data type
router.where('id', router.matchers.number())

// Validate id to be a valid UUID
router.where('id', router.matchers.uuid())

// Validate slug to match a given slug regex: regexr.com/64su0
router.where('slug', router.matchers.slug())
```

### 全局匹配器

路由匹配器可以在路由器实例上全局定义。除非在路由级别显式覆盖，否则全局匹配器将应用于所有路由。

```ts
// title: start/routes.ts
import router from '@adonisjs/core/services/router'

// Global matcher
router.where('id', router.matchers.uuid())

router
  .get('/posts/:id', () => {})
  // Overridden at route level
  .where('id', router.matchers.number())
```

## HTTP 方法

`router.get()` 方法创建一个响应 [GET HTTP 方法](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/GET) 的路由。类似地，您可以使用以下方法为不同的 HTTP 方法注册路由。

```ts
// title: start/routes.ts
import router from '@adonisjs/core/services/router'

// GET method
router.get('users', () => {})

// POST method
router.post('users', () => {})

// PUT method
router.put('users/:id', () => {})

// PATCH method
router.patch('users/:id', () => {})

// DELETE method
router.delete('users/:id', () => {})
```

您可以使用 `router.any()` 方法创建一个响应所有标准 HTTP 方法的路由。

```ts
// title: start/routes.ts
router.any('reports', () => {})
```

最后，您可以使用 `router.route()` 方法为自定义 HTTP 方法创建路由。

```ts
// title: start/routes.ts
router.route('/', ['TRACE'], () => {})
```

## 路由处理程序

路由处理程序通过返回响应或引发异常来中止请求来处理请求。

处理程序可以是内联回调（如本指南中所见）或对控制器方法的引用。

```ts
// title: start/routes.ts
router.post('users', () => {
  // 执行某些操作
})
```

:::note

路由处理程序可以是异步函数，AdonisJS 会自动处理 Promise 解析。

:::

在以下示例中，我们导入 `UsersController` 类并将其绑定到路由。在 HTTP 请求期间，AdonisJS 将使用 IoC 容器创建控制器类的实例并执行 `store` 方法。

另请参阅：[控制器](./controllers.md) 专门指南。

```ts
// title: start/routes.ts
const UsersController = () => import('#controllers/users_controller')

router.post('users', [UsersController, 'store'])
```

## 路由中间件

您可以通过链式调用 `use()` 方法在路由上定义中间件。该方法接受内联回调或对命名中间件的引用。

以下是定义路由中间件的最小示例。我们建议阅读 [中间件](./middleware.md) 专门指南，以探索所有可用选项和中间件的执行流程。

```ts
// title: start/routes.ts
router
  .get('posts', () => {
    console.log('Inside route handler')

    return 'Viewing all posts'
  })
  .use((_, next) => {
    console.log('Inside middleware')
    return next()
  })
```

## 路由标识符

每个路由都有一个唯一的标识符，您可以在应用程序的其他地方使用它来引用该路由。例如，您可以使用 [URL 构建器](#url-builder) 生成路由的 URL，或使用 [`response.redirect()`](./response.md#redirects) 方法重定向到路由。

默认情况下，路由模式就是路由标识符。但是，您可以使用 `route.as` 方法为路由分配一个唯一的、易于记忆的名称。

```ts
// title: start/routes.ts
router.get('users', () => {}).as('users.index')

router.post('users', () => {}).as('users.store')

router.delete('users/:id', () => {}).as('users.delete')
```

现在，您可以在模板中使用路由名称或使用 URL 构建器来构建 URL。

```ts
const url = router.builder().make('users.delete', [user.id])
```

```edge
<form
  method='POST'
  action="{{
    route('users.delete', [user.id], { formAction: 'delete' })
  }}"
></form>
```

## 路由分组

路由组提供了一个便利层，可以批量配置嵌套在组内的路由。您可以使用 `router.group` 方法创建一组路由。

```ts
// title: start/routes.ts
router.group(() => {
  /**
   * 回调内部注册的所有路由
   * 都是周围组的一部分
   */
  router.get('users', () => {})
  router.post('users', () => {})
})
```

路由组可以嵌套在彼此内部，AdonisJS 将根据应用设置的行为合并或覆盖属性。

```ts
// title: start/routes.ts
router.group(() => {
  router.get('posts', () => {})

  router.group(() => {
    router.get('users', () => {})
  })
})
```

### 为组内路由添加前缀

可以使用 `group.prefix` 方法为组内路由的 URI 模式添加前缀。以下示例将为 `/api/users` 和 `/api/payments` URI 模式创建路由。

```ts
// title: start/routes.ts
router
  .group(() => {
    router.get('users', () => {})
    router.get('payments', () => {})
  })
  .prefix('/api')
```

对于嵌套组，前缀将从外层组应用到内层组。以下示例将为 `/api/v1/users` 和 `/api/v1/payments` URI 模式创建路由。

```ts
// title: start/routes.ts
router
  .group(() => {
    router
      .group(() => {
        router.get('users', () => {})
        router.get('payments', () => {})
      })
      .prefix('v1')
  })
  .prefix('api')
```

### 为组内路由命名

类似于为路由模式添加前缀，您也可以使用 `group.as` 方法为组内的路由名称添加前缀。

:::note

组内的路由必须先有名称，然后才能为它们添加前缀。

:::

```ts
// title: start/routes.ts
router
  .group(() => {
    router.get('users', () => {}).as('users.index') // 最终名称 - api.users.index
  })
  .prefix('api')
  .as('api')
```

对于嵌套组，名称将从外层组前缀到内层组。

```ts
// title: start/routes.ts
router
  .group(() => {
    router.get('users', () => {}).as('users.index') // api.users.index

    router
      .group(() => {
        router.get('payments', () => {}).as('payments.index') // api.commerce.payments.index
      })
      .as('commerce')
  })
  .prefix('api')
  .as('api')
```

### 为组内路由应用中间件

您可以使用 `group.use` 方法为组内的路由分配中间件。组中间件在应用于组内单个路由的中间件之前执行。

对于嵌套组，最外层组的中间件将首先运行。换句话说，组会将中间件前置到路由中间件堆栈。

另请参阅：[中间件指南](./middleware.md)

```ts
// title: start/routes.ts
router
  .group(() => {
    router
      .get('posts', () => {})
      .use((_, next) => {
        console.log('logging from route middleware')
        return next()
      })
  })
  .use((_, next) => {
    console.log('logging from group middleware')
    return next()
  })
```

## 为特定域名注册路由

AdonisJS 允许您在特定域名下注册路由。当您有一个映射到多个域名的应用程序，并且希望每个域名有不同的路由时，这很有用。

在以下示例中，我们定义了两组路由：

- 为任何域名/主机名解析的路由。
- 当域名/主机名与预定义的域名值匹配时匹配的路由。

```ts
// title: start/routes.ts
router.group(() => {
  router.get('/users', () => {})
  router.get('/payments', () => {})
})

router
  .group(() => {
    router.get('/articles', () => {})
    router.get('/articles/:id', () => {})
  })
  .domain('blog.adonisjs.com')
```

部署应用程序后，只有当请求的主机名是 `blog.adonisjs.com` 时，具有显式域名的组下的路由才会匹配。

### 动态子域名

您可以使用 `group.domain` 方法指定动态子域名。与路由参数类似，域名的动态段以冒号 `:` 开头。

在以下示例中，`tenant` 段接受任何子域名，您可以使用 `HttpContext.subdomains` 对象访问其值。

```ts
// title: start/routes.ts
router
  .group(() => {
    router.get('users', ({ subdomains }) => {
      return `Listing users for ${subdomains.tenant}`
    })
  })
  .domain(':tenant.adonisjs.com')
```

## 从路由渲染 Edge 视图

如果您有一个仅渲染视图的路由处理程序，您可以使用 `router.on().render()` 方法。这是一个方便的快捷方式，可以渲染视图而无需定义显式的处理程序。

render 方法接受要渲染的 edge 模板的名称。可选地，您可以将模板数据作为第二个参数传递。

:::warning

只有在配置了 [Edge 服务提供程序](../views-and-templates/edgejs.md) 时，`route.on().render()` 方法才会存在。

:::

```ts
// title: start/routes.ts
import router from '@adonisjs/core/services/router'

router.on('/').render('home')
router.on('about').render('about', { title: 'About us' })
router.on('contact').render('contact', { title: 'Contact us' })
```

## 从路由渲染 Inertia 视图

如果您使用 Inertia.js 适配器，您可以使用 `router.on().renderInertia()` 方法渲染 Inertia 视图。这是一个方便的快捷方式，可以渲染视图而无需定义显式的处理程序。

renderInertia 方法接受要渲染的 Inertia 组件的名称。可选地，您可以将组件数据作为第二个参数传递。

:::warning

只有在配置了 [Inertia 服务提供程序](../views-and-templates/inertia.md) 时，`route.on().renderInertia()` 方法才会存在。

:::

```ts
// title: start/routes.ts
import router from '@adonisjs/core/services/router'

router.on('/').renderInertia('home')
router.on('about').renderInertia('about', { title: 'About us' })
router.on('contact').renderInertia('contact', { title: 'Contact us' })
```

## 从路由重定向

如果您要定义一个路由处理程序来将请求重定向到另一个路径或路由，您可以使用 `router.on().redirect()` 或 `router.on().redirectToPath()` 方法。

`redirect` 方法接受路由标识符。而 `redirectToPath` 方法接受静态路径/URL。

```ts
// title: start/routes.ts
import router from '@adonisjs/core/services/router'

// 重定向到路由
router.on('/posts').redirect('/articles')

// 重定向到 URL
router.on('/posts').redirectToPath('https://medium.com/my-blog')
```

### 转发参数

在以下示例中，原始请求中的 `id` 值将用于构建 `/articles/:id` 路由。因此，如果请求是 `/posts/20`，它将被重定向到 `/articles/20`。

```ts
// title: start/routes.ts
import router from '@adonisjs/core/services/router'

router.on('/posts/:id').redirect('/articles/:id')
```

### 显式指定参数

您还可以将路由参数作为第二个参数显式指定。在这种情况下，将忽略当前请求中的参数。

```ts
// title: start/routes.ts
import router from '@adonisjs/core/services/router'

// Always redirect to /articles/1
router.on('/posts/:id').redirect('/articles/:id', {
  id: 1,
})
```

### 带查询字符串

可以在选项对象中定义重定向 URL 的查询字符串。

```ts
// title: start/routes.ts
import router from '@adonisjs/core/services/router'

router.on('/posts').redirect('/articles', {
  qs: {
    limit: 20,
    page: 1,
  },
})
```

## 当前请求路由

当前请求的路由可以使用 [`HttpContext.route`](../concepts/http_context.md#http-context-properties) 属性访问。它包括 **路由模式**、**名称**、**对其中间件存储的引用** 和 **对路由处理程序的引用**。

```ts
// title: start/routes.ts
router.get('payments', ({ route }) => {
  console.log(route)
})
```

您还可以使用 `request.matchesRoute` 方法检查当前请求是否针对特定路由。该方法接受路由 URI 模式或路由名称。

```ts
// title: start/routes.ts
router.get('/posts/:id', ({ request }) => {
  if (request.matchesRoute('/posts/:id')) {
  }
})
```

```ts
// title: start/routes.ts
router
  .get('/posts/:id', ({ request }) => {
    if (request.matchesRoute('posts.show')) {
    }
  })
  .as('posts.show')
```

您还可以匹配多个路由。该方法会在找到第一个匹配项后立即返回 true。

```ts
if (request.matchesRoute(['/posts/:id', '/posts/:id/comments'])) {
  // do something
}
```

## AdonisJS 如何匹配路由

路由按照它们在路由文件中注册的顺序进行匹配。我们从最顶部的路由开始匹配，在第一个匹配的路由处停止。

如果您有两个相似的路由，您必须首先注册最具体的路由。

在以下示例中，URL 为 `/posts/archived` 的请求将由第一个路由（即 `/posts/:id`）处理，因为动态参数 `id` 将捕获 `archived` 值。

```ts
// title: start/routes.ts
import router from '@adonisjs/core/services/router'

router.get('posts/:id', () => {})
router.get('posts/archived', () => {})
```

可以通过重新排序路由来修复此行为，将最具体的路由放在具有动态参数的路由之前。

```ts
// title: start/routes.ts
router.get('posts/archived', () => {})
router.get('posts/:id', () => {})
```

### 处理 404 请求

当没有为当前请求的 URL 找到匹配的路由时，AdonisJS 会引发 404 异常。

要向用户显示 404 页面，您可以在 [全局异常处理程序](./exception_handling.md) 中捕获 `E_ROUTE_NOT_FOUND` 异常并渲染模板。

```ts
// app/exceptions/handler.ts
import { errors } from '@adonisjs/core'
import { HttpContext, ExceptionHandler } from '@adonisjs/core/http'

export default class HttpExceptionHandler extends ExceptionHandler {
  async handle(error: unknown, ctx: HttpContext) {
    if (error instanceof errors.E_ROUTE_NOT_FOUND) {
      return ctx.view.render('errors/404')
    }

    return super.handle(error, ctx)
  }
}
```

## URL 构建器

您可以使用 URL 构建器为应用程序中预定义的路由创建 URL。例如，在 Edge 模板中创建表单操作 URL，或创建 URL 以将请求重定向到另一个路由。

`router.builder` 方法创建 [URL 构建器](https://github.com/adonisjs/http-server/blob/main/src/router/lookup_store/url_builder.ts) 类的实例，您可以使用构建器的流畅 API 查找路由并为其创建 URL。

```ts
// title: start/routes.ts
import router from '@adonisjs/core/services/router'
const PostsController = () => import('#controllers/posts_controller')

router.get('posts/:id', [PostsController, 'show']).as('posts.show')
```

您可以按如下方式生成 `posts.show` 路由的 URL：

```ts
// title: start/routes.ts
import router from '@adonisjs/core/services/router'

router.builder().params([1]).make('posts.show') // /posts/1

router.builder().params([20]).make('posts.show') // /posts/20
```

参数可以指定为位置参数的数组。或者，您可以将它们定义为键值对。

```ts
// title: start/routes.ts
router.builder().params({ id: 1 }).make('posts.show') // /posts/1
```

### 定义查询参数

可以使用 `builder.qs` 方法定义查询参数。该方法接受键值对对象并将其序列化为查询字符串。

```ts
// title: start/routes.ts
router.builder().qs({ page: 1, sort: 'asc' }).make('posts.index') // /posts?page=1&sort=asc
```

查询字符串使用 [qs](https://www.npmjs.com/package/qs) npm 包进行序列化。您可以在 `config/app.ts` 文件中的 `http` 对象下 [配置其设置](https://github.com/adonisjs/http-server/blob/main/src/define_config.ts#L49-L54)。

```ts
// title: config/app.js
http: defineConfig({
  qs: {
    stringify: {
      //
    },
  },
})
```

### 前缀 URL

您可以使用 `builder.prefixUrl` 方法为输出添加基础 URL 前缀。

```ts
// title: start/routes.ts
router
  .builder()
  .prefixUrl('https://blog.adonisjs.com')
  .params({ id: 1 })
  .make('posts.show')
```

### 生成签名 URL

签名 URL 是附加了签名查询字符串的 URL。签名用于验证 URL 在生成后是否被篡改。

例如，您有一个用于取消订阅用户的 URL。该 URL 包含 `userId`，可能如下所示：

```
/unsubscribe/231
```

为了防止有人将用户 ID 从 `231` 更改为其他值，您可以对该 URL 进行签名，并在处理此路由的请求时验证签名。

```ts
// title: start/routes.ts
router
  .get('unsubscribe/:id', ({ request, response }) => {
    if (!request.hasValidSignature()) {
      return response.badRequest('Invalid or expired URL')
    }

    // // 取消订阅
  })
  .as('unsubscribe')
```

您可以使用 `makeSigned` 方法创建签名 URL。

```ts
// title: start/routes.ts
router
  .builder()
  .prefixUrl('https://blog.adonisjs.com')
  .params({ id: 231 })
  // highlight-start
  .makeSigned('unsubscribe')
// highlight-end
```

#### 签名 URL 过期

您可以使用 `expiresIn` 选项生成在给定持续时间后过期的签名 URL。该值可以是毫秒数或时间表达式字符串。

```ts
// title: start/routes.ts
router
  .builder()
  .prefixUrl('https://blog.adonisjs.com')
  .params({ id: 231 })
  // highlight-start
  .makeSigned('unsubscribe', {
    expiresIn: '3 days',
  })
// highlight-end
```

### 禁用路由查找

URL 构建器会对 `make` 和 `makeSigned` 方法提供的路由标识符执行路由查找。

如果您想为 AdonisJS 应用程序外部定义的路由创建 URL，您可以禁用路由查找并将路由模式提供给 `make` 和 `makeSigned` 方法。

```ts
// title: start/routes.ts
router
  .builder()
  .prefixUrl('https://your-app.com')
  .disableRouteLookup()
  .params({ token: 'foobar' })
  .make('/email/verify/:token') // /email/verify/foobar
```

### 为域下的路由生成 URL

您可以使用 `router.builderForDomain` 方法为在特定域下注册的路由生成 URL。该方法接受您在定义路由时使用的路由模式。

```ts
// title: start/routes.ts
import router from '@adonisjs/core/services/router'
const PostsController = () => import('#controllers/posts_controller')

router
  .group(() => {
    router.get('/posts/:id', [PostsController, 'show']).as('posts.show')
  })
  .domain('blog.adonisjs.com')
```

您可以按如下方式为 `blog.adonisjs.com` 域下的 `posts.show` 路由创建 URL：

```ts
// title: start/routes.ts
router
  .builderForDomain('blog.adonisjs.com')
  .params({ id: 1 })
  .make('posts.show')
```

### 在模板中生成 URL

您可以在模板中使用 `route` 和 `signedRoute` 方法，使用 URL 构建器生成 URL。

另请参阅：[Edge 助手参考](../references/edge.md#routesignedroute)

```edge
<a href="{{ route('posts.show', [post.id]) }}">
  View post
</a>
```

```edge
<a href="{{
  signedRoute('unsubscribe', [user.id], {
    expiresIn: '3 days',
    prefixUrl: 'https://blog.adonisjs.com'
  })
}}">
  Unsubscribe
</a>
```

## 扩展路由器

您可以使用宏和 getter 向不同的路由器类添加自定义属性。如果您不熟悉宏的概念，请确保先阅读 [扩展 AdonisJS 指南](../concepts/extending_the_framework.md)。

以下是您可以扩展的类列表：

### Router

[Router 类](https://github.com/adonisjs/http-server/blob/main/src/router/main.ts) 包含用于创建路由、路由组或路由资源的顶级方法。该类的实例通过路由器服务提供。

```ts
import { Router } from '@adonisjs/core/http'

Router.macro('property', function (this: Router) {
  return value
})
Router.getter('propertyName', function (this: Router) {
  return value
})
```

```ts
// title: types/http.ts
declare module '@adonisjs/core/http' {
  export interface Router {
    property: valueType
  }
}
```

### Route

[Route 类](https://github.com/adonisjs/http-server/blob/main/src/router/route.ts) 表示单个路由。调用 `router.get`、`router.post` 和其他类似方法后，将创建 Route 类的实例。

```ts
import { Route } from '@adonisjs/core/http'

Route.macro('property', function (this: Route) {
  return value
})
Router.getter('property', function (this: Route) {
  return value
})
```

```ts
// title: types/http.ts
declare module '@adonisjs/core/http' {
  export interface Route {
    property: valueType
  }
}
```

### RouteGroup

[RouteGroup 类](https://github.com/adonisjs/http-server/blob/main/src/router/group.ts) 表示一组路由。调用 `router.group` 方法后，将创建 RouteGroup 类的实例。

您可以在宏或 getter 实现内部使用 `this.routes` 属性访问组的路由。

```ts
import { RouteGroup } from '@adonisjs/core/http'

RouteGroup.macro('property', function (this: RouteGroup) {
  return value
})
RouteGroup.getter('property', function (this: RouteGroup) {
  return value
})
```

```ts
// title: types/http.ts
declare module '@adonisjs/core/http' {
  export interface RouteGroup {
    property: valueType
  }
}
```

### RouteResource

[RouteResource 类](https://github.com/adonisjs/http-server/blob/main/src/router/resource.ts) 表示资源的一组路由。调用 `router.resource` 方法后，将创建 RouteResource 类的实例。

您可以在宏或 getter 实现内部使用 `this.routes` 属性访问资源的路由。

```ts
import { RouteResource } from '@adonisjs/core/http'

RouteResource.macro('property', function (this: RouteResource) {
  return value
})
RouteResource.getter('property', function (this: RouteResource) {
  return value
})
```

```ts
// title: types/http.ts
declare module '@adonisjs/core/http' {
  export interface RouteResource {
    property: valueType
  }
}
```

### BriskRoute

[BriskRoute 类](https://github.com/adonisjs/http-server/blob/main/src/router/brisk.ts) 表示没有显式处理程序的路由。调用 `router.on` 方法后，将创建 BriskRoute 类的实例。

您可以在宏或 getter 内部调用 `this.setHandler` 方法来分配路由处理程序。

```ts
import { BriskRoute } from '@adonisjs/core/http'

BriskRoute.macro('property', function (this: BriskRoute) {
  return value
})
BriskRouter.getter('property', function (this: BriskRoute) {
  return value
})
```

```ts
// title: types/http.ts
declare module '@adonisjs/core/http' {
  export interface BriskRoute {
    property: valueType
  }
}
```
