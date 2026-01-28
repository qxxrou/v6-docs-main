---
summary: 学习如何在 AdonisJS 中使用 Inertia，通过你喜爱的前端框架创建服务器渲染的应用程序。
---

# Inertia

[Inertia](https://inertiajs.com/) 是一种与框架无关的方式，可以在不具备现代 SPA 复杂性的情况下创建单页应用程序。

它是传统服务器渲染应用程序（使用模板引擎）和现代 SPA（使用客户端路由和状态管理）之间的绝佳中间地带。

使用 Inertia 可以让你通过喜爱的前端框架（Vue.js、React、Svelte 或 Solid.js）创建 SPA，而无需创建单独的 API。

:::codegroup

```ts
// title: app/controllers/users_controller.ts
import type { HttpContext } from '@adonisjs/core/http'

export default class UsersController {
  async index({ inertia }: HttpContext) {
    const users = await User.all()

    return inertia.render('users/index', { users })
  }
}
```

```vue
// title: inertia/pages/users/index.vue
<script setup lang="ts">
  import { Link, Head } from '@inertiajs/vue3'

  defineProps<{
    users: SerializedUser[]
  }>()
</script>

<template>
  <Head title="Users" />

  <div v-for="user in users" :key="user.id">
    <Link :href="`/users/${user.id}`">
      {{ user.name }}
    </Link>
    <div>{{ user.email }}</div>
  </div>
</template>
```

:::

## 安装

:::note
要开始一个新项目并使用 Inertia？请查看 [Inertia 入门套件](https://docs.adonisjs.com/guides/getting-started/installation#inertia-starter-kit)。
:::

从 npm 注册表运行安装包：

:::codegroup

```sh
// title: npm
npm i @adonisjs/inertia
```

:::

完成后，运行以下命令来配置包。

```sh
node ace configure @adonisjs/inertia
```

:::disclosure{title="查看 configure 命令执行的步骤"}

1. 在 `adonisrc.ts` 文件中注册以下服务提供者和命令。

   ```ts
   {
     providers: [
       // ...其他提供者
       () => import('@adonisjs/inertia/inertia_provider'),
     ]
   }
   ```

2. 在 `start/kernel.ts` 文件中注册以下中间件

   ```ts
   router.use([() => import('@adonisjs/inertia/inertia_middleware')])
   ```

3. 创建 `config/inertia.ts` 文件。

4. 将一些存根文件复制到你的应用程序中，帮助你快速开始。每个复制的文件都会根据之前选择的前端框架进行适配。

5. 创建 `./resources/views/inertia_layout.edge` 文件，用于渲染用于启动 Inertia 的 HTML 页面。

6. 创建 `./inertia/css/app.css` 文件，包含用于样式化 `inertia_layout.edge` 视图的内容。

7. 创建 `./inertia/tsconfig.json` 文件，用于区分服务器和客户端的 TypeScript 配置。

8. 创建 `./inertia/app/app.ts` 用于引导 Inertia 和你的前端框架。

9. 创建 `./inertia/pages/home.{tsx|vue|svelte}` 文件来渲染应用程序的首页。

10. 创建 `./inertia/pages/server_error.{tsx|vue|svelte}` 和 `./inertia/pages/not_found.{tsx|vue|svelte}` 文件来渲染错误页面。

11. 在 `vite.config.ts` 文件中添加正确的 vite 插件来编译你的前端框架。

12. 在 `start/routes.ts` 文件中添加一个虚拟路由 `/` 作为示例，使用 Inertia 渲染首页。

13. 根据选择的前端框架安装包。

:::

完成后，你就可以在 AdonisJS 应用程序中使用 Inertia 了。启动开发服务器，访问 `localhost:3333` 即可看到使用你选择的 Inertia 前端框架渲染的首页。

:::note
**阅读 [Inertia 官方文档](https://inertiajs.com/)**。

Inertia 是一个后端无关的库。我们只是创建了一个适配器使其能在 AdonisJS 中工作。在阅读本文档之前，你应该熟悉 Inertia 的概念。

**本文档仅涵盖 AdonisJS 特有的部分。**
:::

## 客户端入口点

如果你使用了 `configure` 或 `add` 命令，包将在 `inertia/app/app.ts` 处创建一个入口点文件，你可以跳过此步骤。

基本上，这个文件将是你的前端应用程序的主要入口点，用于引导 Inertia 和你的前端框架。这个文件应该是由你的根 Edge 模板使用 `@vite` 标签加载的入口点。

:::codegroup

```ts
// title: Vue
import { createApp, h } from 'vue'
import type { DefineComponent } from 'vue'
import { createInertiaApp } from '@inertiajs/vue3'
import { resolvePageComponent } from '@adonisjs/inertia/helpers'

const appName = import.meta.env.VITE_APP_NAME || 'AdonisJS'

createInertiaApp({
  title: (title) => `${title} - ${appName}`,
  resolve: (name) => {
    return resolvePageComponent(
      `../pages/${name}.vue`,
      import.meta.glob<DefineComponent>('../pages/**/*.vue'),
    )
  },
  setup({ el, App, props, plugin }) {
    createApp({ render: () => h(App, props) })
      .use(plugin)
      .mount(el)
  },
})
```

```tsx
// title: React
import { createRoot } from 'react-dom/client'
import { createInertiaApp } from '@inertiajs/react'
import { resolvePageComponent } from '@adonisjs/inertia/helpers'

const appName = import.meta.env.VITE_APP_NAME || 'AdonisJS'

createInertiaApp({
  progress: { color: '#5468FF' },

  title: (title) => `${title} - ${appName}`,

  resolve: (name) => {
    return resolvePageComponent(
      `./pages/${name}.tsx`,
      import.meta.glob('./pages/**/*.tsx'),
    )
  },

  setup({ el, App, props }) {
    const root = createRoot(el)
    root.render(<App {...props} />)
  },
})
```

```ts
// title: Svelte
import { createInertiaApp } from '@inertiajs/svelte'
import { resolvePageComponent } from '@adonisjs/inertia/helpers'

const appName = import.meta.env.VITE_APP_NAME || 'AdonisJS'

createInertiaApp({
  progress: { color: '#5468FF' },

  title: (title) => `${title} - ${appName}`,

  resolve: (name) => {
    return resolvePageComponent(
      `./pages/${name}.svelte`,
      import.meta.glob('./pages/**/*.svelte'),
    )
  },

  setup({ el, App, props }) {
    new App({ target: el, props })
  },
})
```

```ts
// title: Solid
import { render } from 'solid-js/web'
import { createInertiaApp } from 'inertia-adapter-solid'
import { resolvePageComponent } from '@adonisjs/inertia/helpers'

const appName = import.meta.env.VITE_APP_NAME || 'AdonisJS'

createInertiaApp({
  progress: { color: '#5468FF' },

  title: (title) => `${title} - ${appName}`,

  resolve: (name) => {
    return resolvePageComponent(
      `./pages/${name}.tsx`,
      import.meta.glob('./pages/**/*.tsx'),
    )
  },

  setup({ el, App, props }) {
    render(() => <App {...props} />, el)
  },
})
```

:::

这个文件的作用是创建 Inertia 应用并解析页面组件。当你使用 `inertia.render` 时编写的页面组件将被传递给 `resolve` 函数，该函数的作用是返回需要渲染的组件。

## 渲染页面

在配置包时，`inertia_middleware` 已在 `start/kernel.ts` 文件中注册。该中间件负责在 [`HttpContext`](../concepts/http_context.md) 上设置 `inertia` 对象。

要使用 Inertia 渲染视图，请使用 `inertia.render` 方法。该方法接受视图名称和要作为 props 传递给组件的数据。

```ts
// title: app/controllers/home_controller.ts
export default class HomeController {
  async index({ inertia }: HttpContext) {
    // highlight-start
    return inertia.render('home', { user: { name: 'julien' } })
    // highlight-end
  }
}
```

你看到传递给 `inertia.render` 方法的 `home` 了吗？它应该是相对于 `inertia/pages` 目录的组件文件路径。这里我们渲染 `inertia/pages/home.(vue,tsx)` 文件。

你的前端组件将接收 `user` 对象作为 prop：

:::codegroup

```vue
// title: Vue
<script setup lang="ts">
  defineProps<{
    user: { name: string }
  }>()
</script>

<template>
  <p>Hello {{ user.name }}</p>
</template>
```

```tsx
// title: React
export default function Home(props: { user: { name: string } }) {
  return <p>Hello {props.user.name}</p>
}
```

```svelte
// title: Svelte
<script lang="ts">
export let user: { name: string }
</script>

<Layout>
  <p>Hello {user.name}</p>
</Layout>
```

```jsx
// title: Solid
export default function Home(props: { user: { name: string } }) {
  return <p>Hello {props.user.name}</p>
}
```

:::

就这么简单。

:::warning
向前端传递数据时，所有内容都会序列化为 JSON。不要期望传递模型实例、日期或其他复杂对象。
:::

### 根 Edge 模板

根模板是一个常规的 Edge 模板，将在应用程序的首次页面访问时加载。这是你应包含 CSS 和 Javascript 文件的地方，也是你应包含 `@inertia` 标签的地方。典型的根模板如下：

:::codegroup

```edge
// title: Vue
<!DOCTYPE html>
<html>

<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title inertia>AdonisJS x Inertia</title>

  @inertiaHead()
  @vite(['inertia/app/app.ts', `inertia/pages/${page.component}.vue`])
</head>

<body>
  @inertia()
</body>

</html>
```

```edge
// title: React
<!DOCTYPE html>
<html>

<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title inertia>AdonisJS x Inertia</title>

  @inertiaHead()
  @viteReactRefresh()
  @vite(['inertia/app/app.tsx', `inertia/pages/${page.component}.tsx`])
</head>

<body>
  @inertia()
</body>

</html>
```

```edge
// title: Svelte
<!DOCTYPE html>
<html>

<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title inertia>AdonisJS x Inertia</title>

  @inertiaHead()
  @vite(['inertia/app/app.ts', `inertia/pages/${page.component}.svelte`])
</head>

<body>
  @inertia()
</body>

</html>
```

```edge
// title: Solid
<!DOCTYPE html>
<html>

<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title inertia>AdonisJS x Inertia</title>

  @inertiaHead()
  @vite(['inertia/app/app.tsx', `inertia/pages/${page.component}.tsx`])
</head>

<body>
  @inertia()
</body>

</html>
```

:::

你可以在 `config/inertia.ts` 文件中配置根模板路径。默认情况下，它假定你的模板位于 `resources/views/inertia_layout.edge`。

```ts
import { defineConfig } from '@adonisjs/inertia'

export default defineConfig({
  // 根模板的路径
  // 相对于 `resources/views` 目录
  rootView: 'app_root',
})
```

如果需要，你可以向 `rootView` 属性传递一个函数，以动态决定应使用哪个根模板。

```ts
import { defineConfig } from '@adonisjs/inertia'
import type { HttpContext } from '@adonisjs/core/http'

export default defineConfig({
  rootView: ({ request }: HttpContext) => {
    if (request.url().startsWith('/admin')) {
      return 'admin_root'
    }

    return 'app_root'
  },
})
```

### 根模板数据

你可能希望与根 Edge 模板共享数据。例如，为了添加元标题或开放图谱标签。你可以通过使用 `inertia.render` 方法的第 3 个参数来实现：

```ts
// title: app/controllers/posts_controller.ts
export default class PostsController {
  async index({ inertia }: HttpContext) {
    return inertia.render('posts/details', post, {
      // highlight-start
      title: post.title,
      description: post.description,
      // highlight-end
    })
  }
}
```

`title` 和 `description` 现在将可用于根 Edge 模板：

```edge
// title: resources/views/root.edge
<html>
  <title>{{ title }}</title>
  <meta name="description" content="{{ description }}">

  <body>
    @inertia()
  </body>
</html
```

## 重定向

这就是你在 AdonisJS 中应该做的：

```ts
export default class UsersController {
  async store({ response }: HttpContext) {
    await User.create(request.body())

    // 👇 你可以使用标准的 AdonisJS 重定向
    return response.redirect().toRoute('users.index')
  }

  async externalRedirect({ inertia }: HttpContext) {
    // 👇 或者使用 inertia.location 进行外部重定向
    return inertia.location('https://adonisjs.com')
  }
}
```

有关更多信息，请参阅[官方文档](https://inertiajs.com/redirects)。

## 与所有视图共享数据

有时，你可能需要跨多个视图共享相同的数据。例如，我们与所有视图共享当前用户信息。为每个控制器都这样做可能会变得繁琐。幸运的是，我们有两个解决方案。

### `sharedData`

在 `config/inertia.ts` 文件中，你可以定义一个 `sharedData` 对象。该对象允许你定义应与所有视图共享的数据。

```ts
import { defineConfig } from '@adonisjs/inertia'

export default defineConfig({
  sharedData: {
    appName: 'My App', // 👈 这将可用于所有视图
    user: (ctx) => ctx.auth?.user, // 👈 限定到当前请求
  },
})
```

### 从中间件共享

有时，从中间件而不是 `config/inertia.ts` 文件共享数据可能更方便。你可以通过使用 `inertia.share` 方法来实现：

```ts
import type { HttpContext } from '@adonisjs/core/http'
import type { NextFn } from '@adonisjs/core/types/http'

export default class MyMiddleware {
  async handle({ inertia, auth }: HttpContext, next: NextFn) {
    inertia.share({
      appName: 'My App',
      user: (ctx) => ctx.auth?.user,
    })
  }
}
```

## 部分重载和延迟数据评估

首先阅读[官方文档](https://inertiajs.com/partial-reloads)以了解什么是部分重载以及它们如何工作。

关于延迟数据评估，它在 AdonisJS 中的工作方式如下：

```ts
export default class UsersController {
  async index({ inertia }: HttpContext) {
    return inertia.render('users/index', {
      // 首次访问时始终包含。
      // 部分重载时可选包含。
      // 始终评估
      users: await User.all(),

      // 首次访问时始终包含。
      // 部分重载时可选包含。
      // 仅在需要时评估
      users: () => User.all(),

      // 首次访问时从不包含。
      // 部分重载时可选包含。
      // 仅在需要时评估
      users: inertia.optional(() => User.all())
    }),
  }
}
```

## 类型共享

通常，你会希望共享传递给前端页面组件的数据类型。一个简单的方法是使用 `InferPageProps` 类型。

:::codegroup

```ts
// title: app/controllers/users_controller.ts
export class UsersController {
  index() {
    return inertia.render('users/index', {
      users: [
        { id: 1, name: 'julien' },
        { id: 2, name: 'virk' },
        { id: 3, name: 'romain' },
      ],
    })
  }
}
```

```tsx
// title: inertia/pages/users/index.tsx
import { InferPageProps } from '@adonisjs/inertia/types'
import type { UsersController } from '../../controllers/users_controller.ts'

export function UsersPage(
  // 👇 它将根据你传递给 inertia.render 的内容正确键入
  // 在你的控制器中
  props: InferPageProps<UsersController, 'index'>
) {
  return (
    // ...
  )
}
```

:::

如果你使用 Vue，你必须在 `defineProps` 中手动定义每个属性。这是 Vue 的一个烦人限制，有关更多信息，请参阅[此问题](https://github.com/vitejs/vite-plugin-vue/issues/167)。

```vue
<script setup lang="ts">
  import { InferPageProps } from '@adonisjs/inertia/types'

  defineProps<{
    // 👇 你将必须手动定义每个 prop
    users: InferPageProps<UsersController, 'index'>['users']
    posts: InferPageProps<PostsController, 'index'>['posts']
  }>()
</script>
```

### 引用指令

由于你的 Inertia 应用程序是一个单独的 TypeScript 项目（有自己的 `tsconfig.json`），你需要帮助 TypeScript 理解某些类型。我们的许多官方包使用[模块增强](https://www.typescriptlang.org/docs/handbook/declaration-merging.html#module-augmentation)向你的 AdonisJS 项目添加某些类型。

例如，只有当你将 `@adonisjs/auth/initialize_auth_middleware` 导入到你的项目中时，`HttpContext` 上的 `auth` 属性及其类型才可用。现在，问题是我们没有在我们的 Inertia 项目中导入此模块，因此如果你尝试从使用 `auth` 的控制器推断页面属性，那么你可能会收到 TypeScript 错误或无效类型。

要解决此问题，你可以使用[引用指令](https://www.typescriptlang.org/docs/handbook/triple-slash-directives.html#-reference-path-)来帮助 TypeScript 理解某些类型。为此，你需要在 `inertia/app/app.ts` 文件中添加以下行：

```ts
/// <reference path="../../adonisrc.ts" />
```

根据你使用的类型，你可能需要添加其他引用指令，例如对某些也使用模块增强的配置文件的引用。

```ts
/// <reference path="../../adonisrc.ts" />
/// <reference path="../../config/ally.ts" />
/// <reference path="../../config/auth.ts" />
```

### 类型级序列化

关于 `InferPageProps` 的一个重要事项是，它将"在类型级别序列化"你传递的数据。例如，如果你将 `Date` 对象传递给 `inertia.render`，则 `InferPageProps` 的结果类型将是 `string`：

:::codegroup

```ts
// title: app/controllers/users_controller.ts
export default class UsersController {
  async index({ inertia }: HttpContext) {
    const users = [{ id: 1, name: 'John Doe', createdAt: new Date() }]

    return inertia.render('users/index', { users })
  }
}
```

```tsx
// title: inertia/pages/users/index.tsx
import type { InferPageProps } from '@adonisjs/inertia/types'

export function UsersPage(props: InferPageProps<UsersController, 'index'>) {
  props.users
  //     ^? { id: number, name: string, createdAt: string }[]
}
```

:::

这完全合理，因为日期在通过网络以 JSON 形式传递时会序列化为字符串。

### 模型序列化

记住最后一点，另一个重要事项是，如果你将 AdonisJS 模型传递给 `inertia.render`，则 `InferPageProps` 的结果类型将是 `ModelObject`：一个几乎不包含任何信息的类型。这可能是个问题。要解决此问题，你有几个选项：

- 在将模型传递给 `inertia.render` 之前将其转换为简单对象：
- 使用 DTO（数据传输对象）系统在将模型传递给 `inertia.render` 之前将模型转换为简单对象。

:::codegroup

```ts
// title: 转换
class UsersController {
  async edit({ inertia, params }: HttpContext) {
    const user = users.serialize() as {
      id: number
      name: string
    }

    return inertia.render('user/edit', { user })
  }
}
```

```ts
// title: DTO
class UserDto {
  constructor(private user: User) {}

  toJson() {
    return {
      id: this.user.id,
      name: this.user.name,
    }
  }
}

class UsersController {
  async edit({ inertia, params }: HttpContext) {
    const user = await User.findOrFail(params.id)
    return inertia.render('user/edit', { user: new UserDto(user).toJson() })
  }
}
```

:::

现在你的前端组件中将拥有准确的类型。

### 共享属性

要在组件中拥有[共享数据](#与所有视图共享数据)的类型，请确保在 `config/inertia.ts` 文件中执行了模块增强，如下所示：

```ts
// file: config/inertia.ts
const inertiaConfig = defineConfig({
  sharedData: {
    appName: 'My App',
  },
})

export default inertiaConfig

declare module '@adonisjs/inertia/types' {
  export interface SharedProps extends InferSharedProps<typeof inertiaConfig> {
    // 如有必要，你还可以手动添加一些共享属性，
    // 例如那些从中间件共享的属性
    propsSharedFromAMiddleware: number
  }
}
```

另外，确保在 `inertia/app/app.ts` 文件中添加此[引用指令](#引用指令)：

```ts
/// <reference path="../../config/inertia.ts" />
```

完成后，你将能够通过 `InferPageProps` 在组件中访问共享属性。`InferPageProps` 将包含共享属性的类型和 `inertia.render` 传递的属性：

```tsx
// file: inertia/pages/users/index.tsx

import type { InferPageProps } from '@adonisjs/inertia/types'

export function UsersPage(props: InferPageProps<UsersController, 'index'>) {
  props.appName
  //     ^? string
  props.propsSharedFromAMiddleware
  //     ^? number
}
```

如有需要，你可以通过 `SharedProps` 类型仅访问共享属性的类型：

```tsx
import type { SharedProps } from '@adonisjs/inertia/types'

const page = usePage<SharedProps>()
```

## CSRF

如果你为应用程序启用了 [CSRF 保护](../security/securing_ssr_applications.md#csrf-protection)，请在 `config/shield.ts` 文件中启用 `enableXsrfCookie` 选项。

启用此选项将确保在客户端设置 `XSRF-TOKEN` cookie，并在每个请求中发送回服务器。

无需额外配置即可使 Inertia 与 CSRF 保护一起工作。

## 资源版本控制

重新部署应用程序时，用户应始终获得客户端资源的最新版本。这是 Inertia 协议和 AdonisJS 原生支持的功能。

默认情况下，`@adonisjs/inertia` 包将为 `public/assets/manifest.json` 文件计算哈希值，并将其用作资源的版本。

如果你想调整此行为，可以编辑 `config/inertia.ts` 文件。`assetsVersion` 属性定义资源的版本，可以是字符串或函数。

```ts
import { defineConfig } from '@adonisjs/inertia'

export default defineConfig({
  assetsVersion: 'v1',
})
```

阅读[官方文档](https://inertiajs.com/asset-versioning)以获取更多信息。

## SSR

### 启用 SSR

[Inertia 入门套件](../getting_started/installation.md#starter-kits)原生支持服务器端渲染（SSR）。因此，如果你想为应用程序启用 SSR，请确保使用它。

如果你在未启用 SSR 的情况下启动了应用程序，你可以随时按照以下步骤在以后启用它：

#### 添加服务器入口点

我们需要添加一个服务器入口点，它看起来与客户端入口点非常相似。此入口点将在服务器上渲染首次页面访问，而不是在浏览器上。

你必须创建一个 `inertia/app/ssr.ts` 文件，默认导出一个如下所示的函数：

:::codegroup

```ts
// title: Vue
import { createInertiaApp } from '@inertiajs/vue3'
import { renderToString } from '@vue/server-renderer'
import { createSSRApp, h, type DefineComponent } from 'vue'

export default function render(page) {
  return createInertiaApp({
    page,
    render: renderToString,
    resolve: (name) => {
      const pages = import.meta.glob<DefineComponent>('../pages/**/*.vue')
      return pages[`../pages/${name}.vue`]()
    },

    setup({ App, props, plugin }) {
      return createSSRApp({ render: () => h(App, props) }).use(plugin)
    },
  })
}
```

```tsx
// title: React
import ReactDOMServer from 'react-dom/server'
import { createInertiaApp } from '@inertiajs/react'

export default function render(page) {
  return createInertiaApp({
    page,
    render: ReactDOMServer.renderToString,
    resolve: (name) => {
      const pages = import.meta.glob('./pages/**/*.tsx', { eager: true })
      return pages[`./pages/${name}.tsx`]
    },
    setup: ({ App, props }) => <App {...props} />,
  })
}
```

```ts
// title: Svelte
import { createInertiaApp } from '@inertiajs/svelte'
import createServer from '@inertiajs/svelte/server'

export default function render(page) {
  return createInertiaApp({
    page,
    resolve: (name) => {
      const pages = import.meta.glob('./pages/**/*.svelte', { eager: true })
      return pages[`./pages/${name}.svelte`]
    },
  })
}
```

```tsx
// title: Solid
import { hydrate } from 'solid-js/web'
import { createInertiaApp } from 'inertia-adapter-solid'

export default function render(page: any) {
  return createInertiaApp({
    page,
    resolve: (name) => {
      const pages = import.meta.glob('./pages/**/*.tsx', { eager: true })
      return pages[`./pages/${name}.tsx`]
    },
    setup({ el, App, props }) {
      hydrate(() => <App {...props} />, el)
    },
  })
}
```

:::

#### 更新配置文件

前往 `config/inertia.ts` 文件并更新 `ssr` 属性以启用它。另外，如果你使用不同的路径，请指向你的服务器入口点。

```ts
import { defineConfig } from '@adonisjs/inertia'

export default defineConfig({
  // ...
  ssr: {
    enabled: true,
    entrypoint: 'inertia/app/ssr.ts',
  },
})
```

#### 更新 Vite 配置

首先，确保你已注册 `inertia` vite 插件。完成后，如果你使用不同的路径，应在 `vite.config.ts` 文件中更新服务器入口点的路径。

```ts
import { defineConfig } from 'vite'
import inertia from '@adonisjs/inertia/client'

export default defineConfig({
  plugins: [
    inertia({
      ssr: {
        enabled: true,
        entrypoint: 'inertia/app/ssr.ts',
      },
    }),
  ],
})
```

你现在应该能够在服务器上渲染首次页面访问，然后继续使用客户端渲染。

### SSR 允许列表

使用 SSR 时，你可能不希望服务器端渲染所有组件。例如，你正在构建一个由身份验证保护的管理仪表板，因此这些路由没有理由在服务器上渲染。但在同一应用程序上，你可能有一个可以从 SSR 中受益以提高 SEO 的登录页面。

因此，你可以在 `config/inertia.ts` 文件中添加应在服务器上渲染的页面。

```ts
import { defineConfig } from '@adonisjs/inertia'

export default defineConfig({
  ssr: {
    enabled: true,
    pages: ['home'],
  },
})
```

你还可以向 `pages` 属性传递一个函数，以动态决定哪些页面应在服务器上渲染。

```ts
import { defineConfig } from '@adonisjs/inertia'

export default defineConfig({
  ssr: {
    enabled: true,
    pages: (ctx, page) => !page.startsWith('admin'),
  },
})
```

## 测试

有几种方法可以测试你的前端代码：

- 端到端测试。你可以使用[浏览器客户端](https://docs.adonisjs.com/guides/browser-tests)，它是 Japa 和 Playwright 之间的无缝集成。
- 单元测试。我们建议使用适用于前端生态系统的测试工具，特别是 [Vitest](https://vitest.dev)。

最后，你还可以测试你的 Inertia 端点以确保它们返回正确的数据。为此，我们在 Japa 中提供了一些测试助手。

首先，确保在 `test/bootsrap.ts` 文件中配置 `inertiaApiClient` 和 `apiClient` 插件（如果你尚未这样做）：

```ts
// title: tests/bootstrap.ts
import { assert } from '@japa/assert'
import app from '@adonisjs/core/services/app'
import { pluginAdonisJS } from '@japa/plugin-adonisjs'
// highlight-start
import { apiClient } from '@japa/api-client'
import { inertiaApiClient } from '@adonisjs/inertia/plugins/api_client'
// highlight-end

export const plugins: Config['plugins'] = [
  assert(),
  pluginAdonisJS(app),
  // highlight-start
  apiClient(),
  inertiaApiClient(app),
  // highlight-end
]
```

接下来，我们可以使用 `withInertia()` 请求我们的 Inertia 端点，以确保数据以 JSON 格式正确返回。

```ts
test('returns correct data', async ({ client }) => {
  const response = await client.get('/home').withInertia()

  response.assertStatus(200)
  response.assertInertiaComponent('home/main')
  response.assertInertiaProps({ user: { name: 'julien' } })
})
```

让我们看一下可用于测试端点的各种断言：

### `withInertia()`

将 `X-Inertia` 标头添加到请求。它确保数据以 JSON 格式正确返回。

### `assertInertiaComponent()`

检查服务器返回的组件是否是预期的组件。

```ts
test('returns correct data', async ({ client }) => {
  const response = await client.get('/home').withInertia()

  response.assertInertiaComponent('home/main')
})
```

### `assertInertiaProps()`

检查服务器返回的属性是否完全与作为参数传递的属性相同。

```ts
test('returns correct data', async ({ client }) => {
  const response = await client.get('/home').withInertia()

  response.assertInertiaProps({ user: { name: 'julien' } })
})
```

### `assertInertiaPropsContains()`

检查服务器返回的属性是否包含作为参数传递的某些属性。它在底层使用 [`containsSubset`](https://japa.dev/docs/plugins/assert#containssubset)。

```ts
test('returns correct data', async ({ client }) => {
  const response = await client.get('/home').withInertia()

  response.assertInertiaPropsContains({ user: { name: 'julien' } })
})
```

### 附加属性

除此之外，你还可以在 `ApiResponse` 上访问以下属性：

```ts
test('returns correct data', async ({ client }) => {
  const response = await client.get('/home').withInertia()

  // 👇 服务器返回的组件
  console.log(response.inertiaComponent)

  // 👇 服务器返回的属性
  console.log(response.inertiaProps)
})
```

## 常见问题

### 为什么更新前端代码时服务器不断重新加载？

假设你正在使用 React。每当你更新前端代码时，服务器都会重新加载，浏览器也会刷新。你没有从热模块替换（HMR）功能中受益。

你需要从根 `tsconfig.json` 文件中排除 `inertia/**/*` 才能使其工作。

```jsonc
{
  "compilerOptions": {
    // ...
  },
  "exclude": ["inertia/**/*"],
}
```

因为，负责重新启动服务器的 AdonisJS 进程正在监视 `tsconfig.json` 文件中包含的文件。

### 为什么我的生产构建不起作用？

如果你遇到如下错误：
`X [ERROR] Failed to load url inertia/app/ssr.ts (resolved id: inertia/app/ssr.ts). Does the file exist?`

一个常见的问题是你在运行生产构建时忘记设置 `NODE_ENV=production`。

```shell
NODE_ENV=production node build/server.js
```

### `Top-level await is not available...`

如果你遇到如下错误：

```
X [ERROR] Top-level await is not available in the configured target environment ("chrome87", "edge88", "es2020", "firefox78", "safari14" + 2 overrides)

node_modules/@adonisjs/core/build/services/hash.js:15:0:
  15 │ await app.booted(async () => {
     ╵ ~~~~~

```

那么你很可能正在将后端代码导入到前端。仔细查看错误，该错误由 Vite 生成，我们看到它正在尝试编译来自 `node_modules/@adonisjs/core` 的代码。因此，这意味着我们的后端代码最终将出现在前端包中。这可能不是你想要的。

通常，当你尝试与前端共享类型时，会发生此错误。如果这是你想要实现的，请确保始终仅通过 `import type` 而不是 `import` 导入此类型：

```ts
// ✅ 正确
import type { User } from '#models/user'

// ❌ 错误
import { User } from '#models/user'
```
