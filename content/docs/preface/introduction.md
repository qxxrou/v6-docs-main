---
summary: 'AdonisJS 是一个 TypeScript 优先的 Node.js Web 框架。你可以使用它来创建全栈 Web 应用程序或 JSON API 服务器。'
---

# 介绍

::include{template="partials/introduction_cards"}

## 什么是 AdonisJS？

AdonisJS 是一个 TypeScript 优先的 Node.js Web 框架。你可以使用它来创建全栈 Web 应用程序或 JSON API 服务器。

在基础层面上，AdonisJS [为你的应用程序提供结构](../getting_started/folder_structure.md)，配置 [无缝的 TypeScript 开发环境](../concepts/typescript_build_process.md)，为你的后端代码配置 [HMR](../concepts/hmr.md)，并提供大量维护良好且文档详尽的包。

我们设想使用 AdonisJS 的团队 **花更少的时间** 在琐碎的决策上，比如为每个小功能挑选 npm 包、编写胶水代码、争论完美的文件夹结构，**花更多的时间** 提供对业务需求至关重要的现实世界功能。

### 前端无关

AdonisJS 专注于后端，让你选择自己喜欢的前端技术栈。

如果你喜欢保持简单，将 AdonisJS 与 [传统模板引擎](../views-and-templates/introduction.md) 配对，在服务器上生成静态 HTML，为你的前端 Vue/React 应用程序创建 JSON API，或使用 [Inertia](../views-and-templates/inertia.md) 让你最喜欢的前端框架与后端完美协调工作。

AdonisJS 旨在为你提供从头开始创建健壮后端应用程序的电池。无论是发送电子邮件、验证用户输入、执行 CRUD 操作还是验证用户身份。我们都会处理这一切。

### 现代且类型安全

AdonisJS 构建在现代 JavaScript 原语之上。我们使用 ES 模块、Node.js 子路径导入别名、用于执行 TypeScript 源代码的 SWC，以及用于资产打包的 Vite。

此外，TypeScript 在设计框架 API 时发挥了重要作用。例如，AdonisJS 具有：

- [类型安全的事件发射器](../digging_deeper/emitter.md#making-events-type-safe)
- [类型安全的环境变量](../getting_started/environment_variables.md)
- [类型安全的验证库](../basics/validation.md)

### 拥抱 MVC

AdonisJS 拥抱经典的 MVC 设计模式。你首先使用功能性 JavaScript API 定义路由，将它们绑定到控制器，并在控制器内编写处理 HTTP 请求的逻辑。

```ts
// title: start/routes.ts
import router from '@adonisjs/core/services/router'
const PostsController = () => import('#controllers/posts_controller')

router.get('posts', [PostsController, 'index'])
```

控制器可以使用模型从数据库获取数据并渲染视图（又名模板）作为响应。

```ts
// title: app/controllers/posts_controller.ts
import Post from '#models/post'
import type { HttpContext } from '@adonisjs/core/http'

export default class PostsController {
  async index({ view }: HttpContext) {
    const posts = await Post.all()
    return view.render('pages/posts/list', { posts })
  }
}
```

如果你正在构建 API 服务器，你可以用 JSON 响应替换视图层。但是，处理和响应 HTTP 请求的流程保持不变。

```ts
// title: app/controllers/posts_controller.ts
import Post from '#models/post'
import type { HttpContext } from '@adonisjs/core/http'

export default class PostsController {
  async index({ view }: HttpContext) {
    const posts = await Post.all()
    // delete-start
    return view.render('pages/posts/list', { posts })
    // delete-end
    // insert-start
    /**
     * 文章数组将被序列化为 JSON
     * 自动地。
     */
    return posts
    // insert-end
  }
}
```

## 指南假设

AdonisJS 文档是作为参考指南编写的，涵盖了核心团队维护的多个包和模块的用法和 API。

**本指南不会教你如何从头开始构建应用程序**。如果你正在寻找教程，我们推荐从 [Adocasts](https://adocasts.com/) 开始你的旅程。Tom（Adocasts 的创建者）创建了一些高质量的截屏视频，帮助你迈出使用 AdonisJS 的第一步。

话虽如此，文档广泛涵盖了可用模块的用法和框架的内部工作原理。

## 最近发布

以下是最近发布的列表。[点击这里](./releases.md) 查看所有发布。

::include{template="partials/recent_releases"}

## 赞助商

::include{template="partials/sponsors"}
