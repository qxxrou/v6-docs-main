---
summary: Learn how to write authorization checks in your AdonisJS application using the `@adonisjs/bouncer` package.
---

# 授权

您可以使用 `@adonisjs/bouncer` 包在 AdonisJS 应用程序中编写授权检查。Bouncer 提供了一个 JavaScript 优先的 API，用于将授权检查编写为 **能力** 和 **策略**。

能力和策略的目标是将授权逻辑抽象到一个地方，并在代码库的其余部分重用它。

- [能力](#defining-abilities) 被定义为函数，如果您的应用程序有较少且简单的授权检查，那么它可能是一个很好的选择。

- [策略](#defining-policies) 被定义为类，您必须为应用程序中的每个资源创建一个策略。策略还可以受益于 [自动依赖注入](#dependency-injection)。

:::note

Bouncer 不是 RBAC 或 ACL 的实现。相反，它提供了一个低级 API，可以对 AdonisJS 应用程序中的操作进行细粒度控制。

:::

## 安装

使用以下命令安装并配置该包：

```sh
node ace add @adonisjs/bouncer
```

:::disclosure{title="查看添加命令执行的步骤"}

1. 使用检测到的包管理器安装 `@adonisjs/bouncer` 包。

2. 在 `adonisrc.ts` 文件中注册以下服务提供程序和命令。

   ```ts
   {
     commands: [
       // ...其他命令
       () => import('@adonisjs/bouncer/commands')
     ],
     providers: [
       // ...其他提供程序
       () => import('@adonisjs/bouncer/bouncer_provider')
     ]
   }
   ```

3. 创建 `app/abilities/main.ts` 文件来定义和导出能力。

4. 创建 `app/policies/main.ts` 文件来将所有策略导出为一个集合。

5. 在 `middleware` 目录中创建 `initialize_bouncer_middleware`。

6. 在 `start/kernel.ts` 文件中注册以下中间件。

   ```ts
   router.use([() => import('#middleware/initialize_bouncer_middleware')])
   ```

:::

:::tip
**您更倾向于视觉学习吗？** - 查看我们的朋友 Adocasts 提供的 [AdonisJS Bouncer ](https://adocasts.com/series/adonisjs-bouncer) 免费屏幕录像系列。
:::

## Initialize bouncer 中间件

在设置过程中，我们在您的应用程序中创建并注册 `#middleware/initialize_bouncer_middleware` 中间件。初始化中间件负责为当前认证用户创建 [Bouncer](https://github.com/adonisjs/bouncer/blob/main/src/bouncer.ts) 类的实例，并通过 `ctx.bouncer` 属性与请求的其余部分共享它。

此外，我们使用 `ctx.view.share` 方法将相同的 Bouncer 实例与 Edge 模板共享。如果您的应用程序中不使用 Edge，请随时从中间件中删除以下代码行。

:::note

您拥有应用程序的源代码，包括初始设置期间创建的文件。因此，不要犹豫更改它们，使它们与您的应用程序环境一起工作。

:::

```ts
async handle(ctx: HttpContext, next: NextFn) {
  ctx.bouncer = new Bouncer(
    () => ctx.auth.user || null,
    abilities,
    policies
  ).setContainerResolver(ctx.containerResolver)

  // delete-start
  /**
   * 如果不使用 Edge，请删除
   */
  if ('view' in ctx) {
    ctx.view.share(ctx.bouncer.edgeHelpers)
  }
  // delete-end

  return next()
}
```

## 定义能力

能力是通常编写在 `./app/abilities/main.ts` 文件中的 JavaScript 函数。您可以从这个文件中导出多个能力。

在以下示例中，我们使用 `Bouncer.ability` 方法定义了一个名为 `editPost` 的能力。实现回调必须返回 `true` 以授权用户，并返回 `false` 以拒绝访问。

:::note

能力应该总是接受 `User` 作为第一个参数，然后是授权检查所需的其他参数。

:::

```ts
// title: app/abilities/main.ts
import User from '#models/user'
import Post from '#models/post'
import { Bouncer } from '@adonisjs/bouncer'

export const editPost = Bouncer.ability((user: User, post: Post) => {
  return user.id === post.userId
})
```

### 执行授权

定义能力后，您可以使用 `ctx.bouncer.allows` 方法执行授权检查。

Bouncer 会自动将当前登录的用户作为第一个参数传递给能力回调，您必须手动提供其余参数。

```ts
import Post from '#models/post'
// highlight-start
import { editPost } from '#abilities/main'
// highlight-end
import router from '@adonisjs/core/services/router'

router.put('posts/:id', async ({ bouncer, params, response }) => {
  /**
   * 通过 ID 查找帖子，以便对其执行授权检查。
   */
  const post = await Post.findOrFail(params.id)

  /**
   * 使用能力来查看登录用户是否被允许执行该操作。
   */
  // highlight-start
  if (await bouncer.allows(editPost, post)) {
    return '您可以编辑该帖子'
  }
  // highlight-end

  return response.forbidden('您不能编辑该帖子')
})
```

`bouncer.allows` 方法的相反方法是 `bouncer.denies` 方法。您可能更喜欢使用此方法，而不是编写 `if not` 语句。

```ts
if (await bouncer.denies(editPost, post)) {
  response.abort('您不能编辑该帖子', 403)
}
```

### 允许访客用户

默认情况下，Bouncer 拒绝非登录用户的授权检查，而不调用能力回调。

但是，您可能想要定义某些可以与访客用户一起工作的能力。例如，允许访客查看已发布的帖子，但允许帖子的创建者也查看草稿。

您可以使用 `allowGuest` 选项定义允许访客用户的能力。在这种情况下，选项将被定义为第一个参数，回调将是第二个参数。

```ts
export const viewPost = Bouncer.ability(
  // highlight-start
  { allowGuest: true },
  // highlight-end
  (user: User | null, post: Post) => {
    /**
     * 允许所有人访问已发布的帖子
     */
    if (post.isPublished) {
      return true
    }

    /**
     * 访客不能查看未发布的帖子
     */
    if (!user) {
      return false
    }

    /**
     * 帖子的创建者也可以查看未发布的帖子。
     */
    return user.id === post.userId
  },
)
```

### 授权除登录用户之外的用户

如果您想要授权除登录用户之外的用户，您可以使用 `Bouncer` 构造函数为给定用户创建一个新的 bouncer 实例。

```ts
import User from '#models/user'
import { Bouncer } from '@adonisjs/bouncer'

const user = await User.findOrFail(1)
// highlight-start
const bouncer = new Bouncer(user)
// highlight-end

if (await bouncer.allows(editPost, post)) {
}
```

## 定义策略

策略提供了一个抽象层，将授权检查组织为类。建议每个资源创建一个策略。例如，如果您的应用程序有一个 Post 模型，您必须创建一个 `PostPolicy` 类来授权诸如创建或更新帖子之类的操作。

策略存储在 `./app/policies` 目录中，每个文件代表一个策略。您可以通过运行以下命令创建一个新策略。

另请参见：[Make policy 命令](../references/commands.md#makepolicy)

```sh
node ace make:policy post
```

策略类扩展了 [BasePolicy](https://github.com/adonisjs/bouncer/blob/main/src/base_policy.ts) 类，您可以为要执行的授权检查实现方法。在以下示例中，我们定义了授权检查，以 `create`、`edit` 和 `delete` 一个帖子。

```ts
// title: app/policies/post_policy.ts
import User from '#models/user'
import Post from '#models/post'
import { BasePolicy } from '@adonisjs/bouncer'
import { AuthorizerResponse } from '@adonisjs/bouncer/types'

export default class PostPolicy extends BasePolicy {
  /**
   * 每个登录用户都可以创建一个帖子
   */
  create(user: User): AuthorizerResponse {
    return true
  }

  /**
   * 只有帖子创建者可以编辑帖子
   */
  edit(user: User, post: Post): AuthorizerResponse {
    return user.id === post.userId
  }

  /**
   * 只有帖子创建者可以删除帖子
   */
  delete(user: User, post: Post): AuthorizerResponse {
    return user.id === post.userId
  }
}
```

### 执行授权

创建策略后，您可以使用 `bouncer.with` 方法指定要用于授权的策略，然后链式调用 `bouncer.allows` 或 `bouncer.denies` 方法来执行授权检查。

:::note

在 `bouncer.with` 方法之后链式调用的 `allows` 和 `denies` 方法是类型安全的，并且会根据您在策略类上定义的方法显示完成列表。

:::

```ts
import Post from '#models/post'
import PostPolicy from '#policies/post_policy'
import type { HttpContext } from '@adonisjs/core/http'

export default class PostsController {
  async create({ bouncer, response }: HttpContext) {
    // highlight-start
    if (await bouncer.with(PostPolicy).denies('create')) {
      return response.forbidden('不能创建帖子')
    }
    // highlight-end

    //继续控制器逻辑
  }

  async edit({ bouncer, params, response }: HttpContext) {
    const post = await Post.findOrFail(params.id)

    // highlight-start
    if (await bouncer.with(PostPolicy).denies('edit', post)) {
      return response.forbidden('不能编辑帖子')
    }
    // highlight-end

    //继续控制器逻辑
  }

  async delete({ bouncer, params, response }: HttpContext) {
    const post = await Post.findOrFail(params.id)

    // highlight-start
    if (await bouncer.with(PostPolicy).denies('delete', post)) {
      return response.forbidden('不能删除帖子')
    }
    // highlight-end

    //继续控制器逻辑
  }
}
```

### 允许访客用户

[与能力类似](#allowing-guest-users)，策略也可以使用 `@allowGuest` 装饰器为访客用户定义授权检查。例如：

```ts
import User from '#models/user'
import Post from '#models/post'
import { BasePolicy, allowGuest } from '@adonisjs/bouncer'
import type { AuthorizerResponse } from '@adonisjs/bouncer/types'

export default class PostPolicy extends BasePolicy {
  @allowGuest()
  view(user: User | null, post: Post): AuthorizerResponse {
    /**
     * 允许所有人访问已发布的帖子
     */
    if (post.isPublished) {
      return true
    }

    /**
     * 访客不能查看未发布的帖子
     */
    if (!user) {
      return false
    }

    /**
     * 帖子的创建者也可以查看未发布的帖子。
     */
    return user.id === post.userId
  }
}
```

### 策略钩子

您可以在策略类上定义 `before` 和 `after` 模板方法，以在授权检查周围运行操作。一个常见的用例是始终允许或拒绝特定用户的访问。

:::note

无论登录用户如何，都会调用 `before` 和 `after` 方法。因此，请确保处理 `user` 值为 `null` 的情况。

:::

来自 `before` 的响应被解释如下：

- `true` 值将被视为成功授权，并且不会调用操作方法。
- `false` 值将被视为拒绝访问，并且不会调用操作方法。
- 对于 `undefined` 返回值，bouncer 将执行操作方法以执行授权检查。

```ts
export default class PostPolicy extends BasePolicy {
  async before(user: User | null, action: string, ...params: any[]) {
    /**
     * 始终允许管理员用户，无需执行任何检查
     */
    if (user && user.isAdmin) {
      return true
    }
  }
}
```

`after` 方法接收来自操作方法的原始响应，并可以通过返回新值来覆盖先前的响应。来自 `after` 的响应被解释如下：

- `true` 值将被视为成功授权，旧响应将被丢弃。
- `false` 值将被视为拒绝访问，旧响应将被丢弃。
- 对于 `undefined` 返回值，bouncer 将继续使用旧响应。

```ts
import { AuthorizerResponse } from '@adonisjs/bouncer/types'

export default class PostPolicy extends BasePolicy {
  async after(
    user: User | null,
    action: string,
    response: AuthorizerResponse,
    ...params: any[]
  ) {
    if (user && user.isAdmin) {
      return true
    }
  }
}
```

### 依赖注入

策略类是使用 [IoC 容器](../concepts/dependency_injection.md) 创建的；因此，您可以使用 `@inject` 装饰器在策略构造函数中键入提示并注入依赖项。

```ts
import { inject } from '@adonisjs/core'
import { PermissionsResolver } from '#services/permissions_resolver'

// highlight-start
@inject()
// highlight-end
export class PostPolicy extends BasePolicy {
  constructor(
    // highlight-start
    protected permissionsResolver: PermissionsResolver,
    // highlight-end
  ) {
    super()
  }
}
```

如果在 HTTP 请求期间创建了 Policy 类，您还可以在其中注入 [HttpContext](../concepts/http_context.md) 的实例。

```ts
// highlight-start
import { HttpContext } from '@adonisjs/core/http'
// highlight-end
import { PermissionsResolver } from '#services/permissions_resolver'

@inject()
export class PostPolicy extends BasePolicy {
  // highlight-start
  constructor(protected ctx: HttpContext) {
    // highlight-end
    super()
  }
}
```

## 抛出 AuthorizationException

除了 `allows` 和 `denies` 方法外，您还可以使用 `bouncer.authorize` 方法执行授权检查。当检查失败时，此方法将抛出 [AuthorizationException](../references/exceptions.md#e_authorization_failure)。

```ts
router.put('posts/:id', async ({ bouncer, params }) => {
  const post = await Post.findOrFail(params.id)
  // highlight-start
  await bouncer.authorize(editPost, post)
  // highlight-end

  /**
   * 如果没有引发异常，您可以认为用户被允许编辑帖子。
   */
})
```

AdonisJS 将使用以下内容协商规则将 `AuthorizationException` 转换为 `403 - Forbidden` HTTP 响应。

- 带有 `Accept=application/json` 头的 HTTP 请求将接收错误消息数组。每个数组元素将是一个带有 `message` 属性的对象。

- 带有 `Accept=application/vnd.api+json` 头的 HTTP 请求将接收根据 [JSON API](https://jsonapi.org/format/#errors) 规范格式化的错误消息数组。

- 所有其他请求将接收纯文本响应消息。但是，您可以使用 [状态页面](../basics/exception_handling.md#status-pages) 来显示授权错误的自定义错误页面。

您还可以在 [全局异常处理程序](../basics/exception_handling.md) 中自行处理 `AuthorizationException` 错误。

```ts
import app from '@adonisjs/core/services/app'
import { errors } from '@adonisjs/bouncer'
import { HttpContext, ExceptionHandler } from '@adonisjs/core/http'

export default class HttpExceptionHandler extends ExceptionHandler {
  protected debug = !app.inProduction
  protected renderStatusPages = app.inProduction

  async handle(error: unknown, ctx: HttpContext) {
    // highlight-start
    if (error instanceof errors.E_AUTHORIZATION_FAILURE) {
      return ctx.response
        .status(error.status)
        .send(error.getResponseMessage(ctx))
    }
    // highlight-end

    return super.handle(error, ctx)
  }
}
```

## 自定义授权响应

您可以使用 [AuthorizationResponse](https://github.com/adonisjs/bouncer/blob/main/src/response.ts) 类构造错误响应，而不是从能力和策略返回布尔值。

`AuthorizationResponse` 类为您提供了细粒度的控制，可以定义自定义 HTTP 状态码和错误消息。

```ts
import User from '#models/user'
import Post from '#models/post'
import { Bouncer, AuthorizationResponse } from '@adonisjs/bouncer'

export const editPost = Bouncer.ability((user: User, post: Post) => {
  if (user.id === post.userId) {
    return true
  }

  // highlight-start
  return AuthorizationResponse.deny('帖子未找到', 404)
  // highlight-end
})
```

如果您使用 [@adonisjs/i18n](../digging_deeper/i18n.md) 包，您可以使用 `.t` 方法返回本地化响应。翻译消息将在 HTTP 请求期间根据用户的语言用于默认消息。

```ts
export const editPost = Bouncer.ability((user: User, post: Post) => {
  if (user.id === post.userId) {
    return true
  }

  // highlight-start
  return AuthorizationResponse.deny('帖子未找到', 404) // 默认消息
    .t('errors.not_found') // 翻译标识符
  // highlight-end
})
```

### 使用自定义响应构建器

为各个授权检查定义自定义错误消息的灵活性很棒。但是，如果您总是想返回相同的响应，重复相同的代码可能会很麻烦。

因此，您可以按如下方式覆盖 Bouncer 的默认响应构建器。

```ts
import { Bouncer, AuthorizationResponse } from '@adonisjs/bouncer'

Bouncer.responseBuilder = (response: boolean | AuthorizationResponse) => {
  if (response instanceof AuthorizationResponse) {
    return response
  }

  if (response === true) {
    return AuthorizationResponse.allow()
  }

  return AuthorizationResponse.deny('资源未找到', 404).t('errors.not_found')
}
```

## 预注册能力和策略

到目前为止，在本指南中，每当我们想要使用能力或策略时，我们都会显式导入它。但是，一旦您预注册它们，您就可以通过其名称作为字符串来引用能力或策略。

在您的 TypeScript 代码库中，预注册能力和策略可能不如清理导入有用。但是，它们在 Edge 模板中提供了更好的 DX。

查看以下 Edge 模板的代码示例，分别在预注册策略和不预注册策略的情况下。

:::caption{for="error"}
**没有预注册。不，不是超级干净**
:::

```edge
{{-- 首先导入能力 --}}
@let(editPost = (await import('#abilities/main')).editPost)

@can(editPost, post)
  {{-- 可以编辑帖子 --}}
@end
```

:::caption{for="success"}
**预注册**
:::

```edge
{{-- 将能力名称作为字符串引用 --}}
@can('editPost', post)
  {{-- 可以编辑帖子 --}}
@end
```

如果您打开 `initialize_bouncer_middleware.ts` 文件，您会发现我们在创建 Bouncer 实例时已经导入并预注册了能力和策略。

```ts
// highlight-start
import * as abilities from '#abilities/main'
import { policies } from '#policies/main'
// highlight-end

export default class InitializeBouncerMiddleware {
  async handle(ctx, next) {
    ctx.bouncer = new Bouncer(
      () => ctx.auth.user,
      // highlight-start
      abilities,
      policies,
      // highlight-end
    )

    return next()
  }
}
```

### 注意事项

- 如果您决定在代码库的其他部分定义能力，那么请确保在中间件中导入并预注册它们。

- 对于策略，每次运行 `make:policy` 命令时，请确保接受提示以将策略注册到策略集合中。策略集合定义在 `./app/policies/main.ts` 文件中。

  ```ts
  // title: app/policies/main.ts
  export const policies = {
    PostPolicy: () => import('#policies/post_policy'),
    CommentPolicy: () => import('#policies/comment_policy'),
  }
  ```

### 引用预注册的能力和策略

在以下示例中，我们摆脱了导入并通过其名称引用能力和策略。请注意 **基于字符串的 API 也是类型安全的**，但您的代码编辑器的 "转到定义" 功能可能不起作用。

```ts
// title: 能力使用示例
// delete-start
import { editPost } from '#abilities/main'
// delete-end

router.put('posts/:id', async ({ bouncer, params, response }) => {
  const post = await Post.findOrFail(params.id)

  // delete-start
  if (await bouncer.allows(editPost, post)) {
  // delete-end
  // insert-start
  if (await bouncer.allows('editPost', post)) {
  // insert-end
    return '您可以编辑帖子'
  }
})
```

```ts
// title: 策略使用示例
// delete-start
import PostPolicy from '#policies/post_policy'
// delete-end

export default class PostsController {
  async create({ bouncer, response }: HttpContext) {
    // delete-start
    if (await bouncer.with(PostPolicy).denies('create')) {
    // delete-end
    // insert-start
    if (await bouncer.with('PostPolicy').denies('create')) {
    // insert-end
      return response.forbidden('不能创建帖子')
    }

    //Continue with the controller logic
  }
}
```

## Edge 模板中的授权检查

在 Edge 模板中执行授权检查之前，请确保 [预注册能力和策略](#pre-registering-abilities-and-policies)。完成后，您可以使用 `@can` 和 `@cannot` 标签执行授权检查。

这些标签接受 `ability` 名称或 `policy.method` 名称作为第一个参数，然后是能力或策略接受的其余参数。

```edge
// title: 与能力一起使用
@can('editPost', post)
  {{-- 可以编辑帖子 --}}
@end

@cannot('editPost', post)
  {{-- 不能编辑帖子 --}}
@end
```

```edge
// title: 与策略一起使用
@can('PostPolicy.edit', post)
  {{-- 可以编辑帖子 --}}
@end

@cannot('PostPolicy.edit', post)
  {{-- 不能编辑帖子 --}}
@end
```

## 事件

请查看 [事件参考指南](../references/events.md#authorizationfinished) 以查看 `@adonisjs/bouncer` 包分发的事件列表。
