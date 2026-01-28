---
summary: 学习如何在 AdonisJS 应用程序中使用 Vite 打包前端资源。
---

# Vite

AdonisJS 使用 [Vite](https://vitejs.dev/) 来打包应用程序的前端资源。我们提供了一个官方集成，它执行所有将 Vite 与 AdonisJS 等后端框架集成所需的繁重工作。它包括：

- 将 Vite 开发服务器嵌入到 AdonisJS 中。
- 一个专用的 Vite 插件来简化配置选项。
- Edge 辅助函数和标签，用于生成由 Vite 处理的资源的 URL。
- 访问 [Vite 运行时 API](https://vitejs.dev/guide/api-vite-runtime.html#vite-runtime-api) 以执行服务器端渲染（SSR）。

Vite 嵌入在 AdonisJS 开发服务器内部，每个应该由 Vite 处理的请求都会通过 AdonisJS 中间件代理到它。这使我们能够直接访问 Vite 的运行时 API 来执行服务器端渲染（SSR）并管理单个开发服务器。这也意味着资源直接由 AdonisJS 提供，而不是由单独的进程提供。

:::tip
仍在使用 @adonisjs/vite 2.x 吗？[查看迁移指南](https://github.com/adonisjs/vite/releases/tag/v3.0.0)以升级到最新版本。
:::

## 安装

首先，确保至少安装了以下版本的 AdonisJS：

- `@adonisjs/core`：6.9.1 或更高版本
- `@adonisjs/assembler`：7.7.0 或更高版本

然后安装并配置 `@adonisjs/vite` 包。下面的命令安装包和 `vite`，并通过创建必要的配置文件来配置项目。

```sh
// title: npm
node ace add @adonisjs/vite
```

:::disclosure{title="查看 configure 命令执行的步骤"}

1. 在 `adonisrc.ts` 文件中注册以下服务提供者。

   ```ts
   {
     providers: [
       // ...其他提供者
       () => import('@adonisjs/vite/vite_provider'),
     ]
   }
   ```

2. 创建 `vite.config.ts` 和 `config/vite.ts` 配置文件。

3. 创建前端入口点文件，即 `resources/js/app.js`。

:::

完成后，将以下内容添加到您的 `adonisrc.ts` 文件中。

```ts
import { defineConfig } from '@adonisjs/core/build/standalone'

export default defineConfig({
  // highlight-start
  assetsBundler: false,
  hooks: {
    onBuildStarting: [() => import('@adonisjs/vite/build_hook')],
  },
  // highlight-end
})
```

`assetsBundler` 属性设置为 `false` 以关闭由 AdonisJS Assembler 完成的资源打包器管理。

`hooks` 属性注册 `@adonisjs/vite/build_hook` 以执行 Vite 构建过程。有关更多信息，请参见 [Assembler 钩子](../concepts/assembler_hooks.md)。

## 配置

设置过程会创建两个配置文件。`vite.config.ts` 文件用于配置 Vite 打包器，`config/vite.ts` 由 AdonisJS 在后端使用。

### Vite 配置文件

`vite.config.ts` 文件是 Vite 使用的常规配置文件。根据您的项目需求，您可以在此文件中安装并注册额外的 Vite 插件。

默认情况下，`vite.config.ts` 文件使用 AdonisJS 插件，该插件接受以下选项。

```ts
// title: vite.config.ts
import { defineConfig } from 'vite'
import adonisjs from '@adonisjs/vite/client'

export default defineConfig({
  plugins: [
    adonisjs({
      entrypoints: ['resources/js/app.js'],
      reload: ['resources/views/**/*.edge'],
    }),
  ],
})
```

<dl>

<dt>
entrypoints
</dt>

<dd>

`entrypoints` 指的是前端代码库的入口点文件。通常，它将是一个带有额外导入的 JavaScript 或 TypeScript 文件。每个入口点将产生一个单独的输出包。

此外，如果需要，您可以定义多个入口点。例如，一个面向用户的应用程序的入口点和另一个用于管理面板的入口点。

</dd>

<dt>
buildDirectory
</dt>

<dd>

`buildDirectory` 选项定义了输出目录的相对路径。该选项值作为 [`build.outDir`](https://vitejs.dev/config/build-options.html#build-outdir) 选项提供给 Vite。

如果您决定更改默认值，请确保同时更新 `config/vite.ts` 文件中的 `buildDirectory` 路径。

**默认值：public/assets**

</dd>

<dt>
reload
</dt>

<dd>

它包含一个 glob 模式数组，用于在文件更改时监视并重新加载浏览器。默认情况下，我们监视 Edge 模板。但是，您也可以配置额外的模式。

</dd>

<dt>
assetsUrl
</dt>

<dd>

它包含在生产环境中为资源生成链接时要添加前缀的 URL。如果您将 Vite 输出上传到 CDN，则此属性的值应为 CDN 服务器 URL。

确保更新后端配置以使用相同的 `assetsUrl` 值。

</dd>
</dl>

---

### AdonisJS 配置文件

AdonisJS 在后端使用 `config/vite.ts` 文件来了解 Vite 构建过程的输出路径。

```ts
// title: config/vite.ts
import { defineConfig } from '@adonisjs/vite'

const viteBackendConfig = defineConfig({
  buildDirectory: 'public/assets',
  assetsUrl: '/assets',
})

export default viteBackendConfig
```

<dl>

<dt>
buildDirectory
</dt>

<dd>

它包含 Vite 构建输出目录的路径。如果您在 `vite.config.ts` 文件中更改默认值，也必须更新此后端配置。

</dd>

<dt>
assetsUrl
</dt>

<dd>

在生产环境中为资源生成链接时要添加前缀的 URL。如果您将 Vite 输出上传到 CDN，则此属性的值应为 CDN 服务器 URL。

</dd>

<dt>
scriptAttributes
</dt>

<dd>

您可以使用 `scriptAttributes` 属性在使用 `@vite` 标签生成的脚本标签上设置属性。这些属性是键值对的集合。

```ts
// title: config/vite.ts
defineConfig({
  scriptAttributes: {
    defer: true,
    async: true,
  },
})
```

</dd>

<dt>
styleAttributes
</dt>

<dd>

您可以使用 `styleAttributes` 属性在使用 `@vite` 标签生成的链接标签上设置属性。这些属性是键值对的集合。

```ts
// title: config/vite.ts
defineConfig({
  styleAttributes: {
    'data-turbo-track': 'reload',
  },
})
```

您还可以通过为 `styleAttributes` 选项分配函数来有条件地应用属性。

```ts
defineConfig({
  styleAttributes: ({ src, url }) => {
    if (src === 'resources/css/admin.css') {
      return {
        'data-turbo-track': 'reload',
      }
    }
  },
})
```

</dd>

</dl>

## 前端资源的文件夹结构

从技术上讲，AdonisJS 不强制执行任何用于存储前端资源的文件夹结构。您可以按自己喜欢的方式组织它们。

但是，我们建议将前端资源存储在 `resources` 文件夹中，每个资源类都在其子目录中。
