---
summary: Learn about HTTP controllers in AdonisJS and how to organize route handlers inside dedicated files.
---

# Controllers

HTTP 控制器提供了一个抽象层，可以将路由处理程序组织在专门的文件中。您不必在路由文件中表达所有请求处理逻辑，而是将其移至控制器类中。

控制器存储在 `./app/controllers` 目录中，每个控制器都表示为一个普通的 JavaScript 类。您可以通过运行以下命令创建一个新控制器。

另见：[Make controller command](../references/commands.md#makecontroller)

```sh
node ace make:controller users
```

新创建的控制器会使用 `class` 声明进行脚手架搭建，您可以在其中手动创建方法。对于此示例，让我们创建一个 `index` 方法并返回一个用户数组。

```ts
// title: app/controllers/users_controller.ts
export default class UsersController {
  index() {
    return [
      {
        id: 1,
        username: 'virk',
      },
      {
        id: 2,
        username: 'romain',
      },
    ]
  }
}
```

最后，让我们将此控制器绑定到路由。我们将使用 `#controllers` 别名导入控制器。这些别名是使用 [Node.js 的子路径导入功能](../getting_started/folder_structure.md#the-sub-path-imports) 定义的。

```ts
// title: start/routes.ts
import router from '@adonisjs/core/services/router'
const UsersController = () => import('#controllers/users_controller')

router.get('users', [UsersController, 'index'])
```

您可能已经注意到，我们没有创建控制器类的实例，而是直接将其传递给路由。这样做允许 AdonisJS：

- 为每个请求创建一个新的控制器实例。
- 并使用 [IoC 容器](../concepts/dependency_injection.md) 构造类，这允许您利用自动依赖注入。

您还可以注意到，我们使用函数懒加载控制器。

:::warning

使用 [HMR](../concepts/hmr.md) 时需要懒加载控制器。

:::

随着代码库的增长，您会注意到它开始影响应用程序的启动时间。一个常见原因是在路由文件中导入所有控制器。

由于控制器处理 HTTP 请求，它们通常会导入其他模块，如模型、验证器或第三方包。结果，您的路由文件成为导入整个代码库的中心点。

懒加载非常简单，只需将导入语句移到函数后面并使用动态导入即可。

:::tip

您可以使用我们的 [ESLint 插件](https://github.com/adonisjs/tooling-config/tree/main/packages/eslint-plugin) 来强制并自动将标准控制器导入转换为惰性动态导入。

:::

### Using magic strings

另一种懒加载控制器的方法是将控制器及其方法引用为字符串。我们称其为魔法字符串，因为字符串本身没有意义，只是路由器使用它来查找控制器并在后台导入它。

在下面的示例中，我们在路由文件中没有任何导入语句，而是将控制器导入路径 + 方法作为字符串绑定到路由。

```ts
// title: start/routes.ts
import router from '@adonisjs/core/services/router'

router.get('users', '#controllers/users_controller.index')
```

魔法字符串的唯一缺点是它们不是类型安全的。如果您在导入路径中输入错误，编辑器不会给您任何反馈。

从好的方面来说，魔法字符串可以清理路由文件中因导入语句而产生的所有视觉混乱。

使用魔法字符串是主观的，您可以决定是否个人或作为团队使用它们。

## 单动作控制器

AdonisJS 提供了一种定义单动作控制器的方法。这是一种将功能封装到名称明确的类中的有效方法。为此，您需要在控制器内定义一个 handle 方法。

```ts
// title: app/controllers/register_newsletter_subscription_controller.ts
export default class RegisterNewsletterSubscriptionController {
  handle() {
    // ...
  }
}
```

然后，您可以使用以下方式在路由上引用控制器。

```ts
// title: start/routes.ts
router.post('newsletter/subscriptions', [
  RegisterNewsletterSubscriptionController,
])
```

## HTTP 上下文

控制器方法接收 [HttpContext](../concepts/http_context.md) 类的实例作为第一个参数。

```ts
// title: app/controllers/users_controller.ts
import type { HttpContext } from '@adonisjs/core/http'

export default class UsersController {
  index(context: HttpContext) {
    // ...
  }
}
```

## 依赖注入

控制器类使用 [IoC 容器](../concepts/dependency_injection.md) 实例化；因此，您可以在控制器构造函数或控制器方法中键入提示依赖项。

假设您有一个 `UserService` 类，您可以按如下方式将其实例注入控制器：

```ts
// title: app/services/user_service.ts
export class UserService {
  all() {
    // return users from db
  }
}
```

```ts
// title: app/controllers/users_controller.ts
import { inject } from '@adonisjs/core'
import UserService from '#services/user_service'

@inject()
export default class UsersController {
  constructor(private userService: UserService) {}

  index() {
    return this.userService.all()
  }
}
```

### 方法注入

您可以使用 [方法注入](../concepts/dependency_injection.md#using-method-injection) 将 `UserService` 的实例直接注入控制器方法中。在这种情况下，您必须在方法名称上应用 `@inject` 装饰器。

传递给控制器方法的第一个参数始终是 [`HttpContext`](../concepts/http_context.md)。因此，您必须将 `UserService` 作为第二个参数进行类型提示。

```ts
// title: app/controllers/users_controller.ts
import { inject } from '@adonisjs/core'
import { HttpContext } from '@adonisjs/core/http'

import UserService from '#services/user_service'

export default class UsersController {
  @inject()
  index(ctx: HttpContext, userService: UserService) {
    return userService.all()
  }
}
```

### 依赖树

依赖项的自动解析不仅限于控制器。任何注入控制器内的类也可以键入提示依赖项，IoC 容器将为您构建依赖项树。

例如，让我们修改 `UserService` 类以接受 [HttpContext](../concepts/http_context.md) 的实例作为构造函数依赖项。

```ts
// title: app/services/user_service.ts
import { inject } from '@adonisjs/core'
import { HttpContext } from '@adonisjs/core/http'

@inject()
export class UserService {
  constructor(private ctx: HttpContext) {}

  all() {
    console.log(this.ctx.auth.user)
    // return users from db
  }
}
```

进行此更改后，`UserService` 将自动接收 `HttpContext` 类的实例。此外，控制器中不需要进行任何更改。

## Resource-driven controllers

对于传统的 [RESTful](https://en.wikipedia.org/wiki/Representational_state_transfer) 应用程序，控制器应该只设计用于管理单个资源。资源通常是应用程序中的一个实体，如 **User 资源**或 **Post 资源**。

让我们以 Post 资源为例，定义处理其 CRUD 操作的端点。我们将首先创建一个控制器。

您可以使用 `make:controller` ace 命令为资源创建控制器。`--resource` 标志会为控制器搭建以下方法：

```sh
node ace make:controller posts --resource
```

```ts
// title: app/controllers/posts_controller.ts
import type { HttpContext } from '@adonisjs/core/http'

export default class PostsController {
  /**
   * Return list of all posts or paginate through
   * them
   */
  async index({}: HttpContext) {}

  /**
   * Render the form to create a new post.
   *
   * Not needed if you are creating an API server.
   */
  async create({}: HttpContext) {}

  /**
   * Handle form submission to create a new post
   */
  async store({ request }: HttpContext) {}

  /**
   * Display a single post by id.
   */
  async show({ params }: HttpContext) {}

  /**
   * Render the form to edit an existing post by its id.
   *
   * Not needed if you are creating an API server.
   */
  async edit({ params }: HttpContext) {}

  /**
   * Handle the form submission to update a specific post by id
   */
  async update({ params, request }: HttpContext) {}

  /**
   * Handle the form submission to delete a specific post by id.
   */
  async destroy({ params }: HttpContext) {}
}
```

接下来，让我们使用 `router.resource` 方法将 `PostsController` 绑定到资源路由。该方法接受资源名称作为第一个参数，控制器引用作为第二个参数。

```ts
// title: start/routes.ts
import router from '@adonisjs/core/services/router'
const PostsController = () => import('#controllers/posts_controller')

router.resource('posts', PostsController)
```

以下是由 `resource` 方法注册的路由列表。您可以通过运行 `node ace list:routes` 命令查看此列表。

![](./post_resource_routes_list.png)

### 嵌套资源

嵌套资源可以通过使用点 `.` 符号分隔的父资源名称和子资源名称来创建。

在下面的示例中，我们为嵌套在 `posts` 资源下的 `comments` 资源创建路由。

```ts
router.resource('posts.comments', CommentsController)
```

![](./post_comments_resource_routes_list.png)

### 浅层资源

使用嵌套资源时，子资源的路由始终以父资源名称及其 id 为前缀。例如：

- `/posts/:post_id/comments` 路由显示给定帖子的所有评论列表。
- `/posts/:post_id/comments/:id` 路由显示单个评论的 id。

第二个路由中 `/posts/:post_id` 的存在是无关紧要的，因为您可以通过其 id 查找评论。

浅资源通过保持 URL 结构扁平（尽可能）来注册其路由。这次，让我们将 `posts.comments` 注册为浅资源。

```ts
router.shallowResource('posts.comments', CommentsController)
```

![](./shallow_routes_list.png)

### 命名资源路由

使用 `router.resource` 方法创建的路由以资源名称和控制器动作为名。首先，我们将资源名称转换为蛇形命名法，并使用点 `.` 分隔符连接操作名称。

| Resource         | Action name | Route name               |
| ---------------- | ----------- | ------------------------ |
| posts            | index       | `posts.index`            |
| userPhotos       | index       | `user_photos.index`      |
| group-attributes | show        | `group_attributes.index` |

您可以使用 `resource.as` 方法重命名所有路由的前缀。在下面的示例中，我们将 `group_attributes.index` 路由名称重命名为 `attributes.index`。

```ts
// title: start/routes.ts
router.resource('group-attributes', GroupAttributesController).as('attributes')
```

提供给 `resource.as` 方法的前缀会转换为蛇形命名法。如果需要，您可以关闭转换，如下所示：

```ts
// title: start/routes.ts
router
  .resource('group-attributes', GroupAttributesController)
  .as('groupAttributes', false)
```

### 仅注册API路由

创建 API 服务器时，用于创建和更新资源的表单由前端客户端或移动应用程序渲染。因此，为这些端点创建路由是多余的。

您可以使用 `resource.apiOnly` 方法删除 `create` 和 `edit` 路由。结果，只会创建五个路由。

```ts
// title: start/routes.ts
router.resource('posts', PostsController).apiOnly()
```

### 仅注册特定路由

要仅注册特定路由，您可以使用 `resource.only` 或 `resource.except` 方法。

`resource.only` 方法接受一个操作名称数组，并删除除了提到的路由之外的所有其他路由。在下面的示例中，仅会注册 `index`、`store` 和 `destroy` 操作的路由。

```ts
// title: start/routes.ts
router.resource('posts', PostsController).only(['index', 'store', 'destroy'])
```

`resource.except` 方法与 `only` 方法相反，注册除了提到的路由之外的所有路由。

```ts
// title: start/routes.ts
router.resource('posts', PostsController).except(['destroy'])
```

### 重命名资源参数

由 `router.resource` 方法生成的路由使用 `id` 作为参数名称。例如，`GET /posts/:id` 用于查看单个帖子，`DELETE /post/:id` 用于删除帖子。

您可以使用 `resource.params` 方法将参数从 `id` 重命名为其他名称。

```ts
// title: start/routes.ts
router.resource('posts', PostsController).params({ posts: 'post' })
```

上述更改将生成以下路由 _(显示部分列表)_。

| HTTP method | Route               | Controller method |
| ----------- | ------------------- | ----------------- |
| GET         | `/posts/:post`      | show              |
| GET         | `/posts/:post/edit` | edit              |
| PUT         | `/posts/:post`      | update            |
| DELETE      | `/posts/:post`      | destroy           |

使用嵌套资源时，您也可以重命名参数。

```ts
// title: start/routes.ts
router.resource('posts.comments', PostsController).params({
  posts: 'post',
  comments: 'comment',
})
```

### 为资源路由分配中间件

您可以使用 `resource.use` 方法将中间件分配给资源注册的路由。该方法接受一个操作名称数组和要分配给它们的中间件。例如：

```ts
// title: start/routes.ts
import router from '@adonisjs/core/services/router'
import { middleware } from '#start/kernel'

router
  .resource('posts')
  .use(['create', 'store', 'update', 'destroy'], middleware.auth())
```

您可以使用通配符 (\*) 关键字将中间件分配给所有路由。

```ts
// title: start/routes.ts
router.resource('posts').use('*', middleware.auth())
```

最后，您可以多次调用 `.use` 方法来分配多个中间件。例如：

```ts
// title: start/routes.ts
router
  .resource('posts')
  .use(['create', 'store', 'update', 'destroy'], middleware.auth())
  .use(['update', 'destroy'], middleware.someMiddleware())
```
