---
summary: 了解 AdonisJS 官方包为 Edge 模板引擎提供的辅助函数和标签。
---

# Edge 辅助函数和标签

在本指南中，我们将了解 AdonisJS 官方包为 Edge 提供的**辅助函数和标签**。本指南未涵盖 Edge 自带的辅助函数，必须参考 [Edge](https://edgejs.dev/docs/helpers) 文档了解这些内容。

## request

对正在进行的 [HTTP 请求](../basics/request.md) 实例的引用。仅当使用 `ctx.view.render` 方法渲染模板时，该属性才可用。

```edge
{{ request.url() }}
{{ request.input('signature') }}
```

## route/signedRoute

使用 [URL 构建器](../basics/routing.md#url-builder) 为路由创建 URL 的辅助函数。与 URL 构建器不同，视图辅助函数没有流畅的 API，并接受以下参数。

<table>
    <tr>
        <td>位置</td>
        <td>描述</td>
    </tr>
    <tr>
        <td>第1个</td>
        <td>路由标识符或路由模式</td>
    </tr>
    <tr>
        <td>第2个</td>
        <td>路由参数定义为数组或对象。</td>
    </tr>
    <tr>
        <td>第3个</td>
        <td>
          <p>带有以下属性的选项对象。</p>
          <ul>
            <li><code>qs</code>: 将查询字符串参数定义为对象。</li>
            <li><code>domain</code>: 在特定域下搜索路由。</li>
            <li><code>prefixUrl</code>: 为输出添加 URL 前缀。</li>
            <li><code>disableRouteLookup</code>: 启用/禁用路由查找。</li>
          </ul>
        </td>
    </tr>
</table>

```edge
<a href="{{ route('posts.show', [post.id]) }}">
  查看帖子
</a>
```

```edge
<a href="{{
  signedRoute('unsubscribe', [user.id], {
    expiresIn: '3 days',
    prefixUrl: 'https://blog.adonisjs.com'
  })
}}">
 取消订阅
</a>
```

## app

对 [应用程序实例](../concepts/application.md) 的引用。

```edge
{{ app.getEnvironment() }}
```

## config

在 Edge 模板中引用配置值的[辅助函数](../getting_started/configuration.md#reading-config-inside-edge-templates)。您可以使用 `config.has` 方法检查键的值是否存在。

```edge
@if(config.has('app.appUrl'))
  <a href="{{ config('app.appUrl') }}"> 首页 </a>
@else
  <a href="/"> 首页 </a>
@end
```

## session

[会话对象](../basics/session.md#reading-and-writing-data) 的只读副本。您无法在 Edge 模板中修改会话数据。仅当使用 `ctx.view.render` 方法渲染模板时，`session` 属性才可用。

```edge
帖子浏览量: {{ session.get(`post.${post.id}.visits`) }}
```

## flashMessages

[会话 flash 消息](../basics/session.md#flash-messages) 的只读副本。仅当使用 `ctx.view.render` 方法渲染模板时，`flashMessages` 属性才可用。

```edge
@if(flashMessages.has('inputErrorsBag.title'))
  <p>{{ flashMessages.get('inputErrorsBag.title') }}</p>
@end

@if(flashMessages.has('notification'))
  <div class="notification {{ flashMessages.get('notification').type }}">
    {{ flashMessages.get('notification').message }}
  </div>
@end
```

## old

`old` 方法是 `flashMessages.get` 方法的简写。

```edge
<input
  type="text"
  name="email"
  value="{{ old('name') || '' }}"
/>
```

## t

`t` 方法由 `@adonisjs/i18n` 包提供，用于使用 [i18n 类](../digging_deeper/i18n.md#resolving-translations) 显示翻译。该方法接受翻译键标识符、消息数据和回退消息作为参数。

```edge
<h1> {{ t('messages.greeting') }} </h1>
```

## i18n

使用应用程序默认语言环境配置的 I18n 类实例的引用。但是，[`DetectUserLocaleMiddleware`](../digging_deeper/i18n.md#detecting-user-locale-during-an-http-request) 会用为当前 HTTP 请求语言环境创建的实例覆盖此属性。

```edge
{{ i18n.formatCurrency(200, { currency: 'USD' }) }}
```

## auth

对 [ctx.auth](../concepts/http_context.md#http-context-properties) 属性的引用，由 [InitializeAuthMiddleware](https://github.com/adonisjs/auth/blob/main/src/auth/middleware/initialize_auth_middleware.ts#L14) 共享。您可以使用此属性访问有关已登录用户的信息。

```edge
@if(auth.isAuthenticated)
  <p> {{ auth.user.email }} </p>
@end
```

如果您在公共页面（不受 auth 中间件保护）上显示已登录用户信息，那么您可能需要先静默检查用户是否已登录。

```edge
{{-- 检查用户是否已登录 --}}
@eval(await auth.use('web').check())

@if(auth.use('web').isAuthenticated)
  <p> {{ auth.use('web').user.email }} </p>
@end
```

## asset

解析由 Vite 处理的资源的 URL。了解有关 [在 Edge 模板中引用资源](../basics/vite.md#referencing-assets-inside-edge-templates) 的更多信息。

```edge
<img src="{{ asset('resources/images/hero.jpg') }}" />
```

## embedImage / embedImageData

`embedImage` 和 `embedImageData` 辅助函数由 [邮件](../digging_deeper/mail.md#embedding-images) 包添加，仅在渲染模板发送电子邮件时可用。

```edge
<img src="{{
  embedImage(app.makePath('assets/hero.jpg'))
}}" />
```

## @flashMessage

`@flashMessage` 标签为按条件读取给定键的 flash 消息提供了更好的 DX。

:::caption{for="error"}
**而不是编写条件语句**
:::

```edge
@if(flashMessages.has('notification'))
  <div class="notification {{ flashMessages.get('notification').type }}">
    {{ flashMessages.get('notification').message }}
  </div>
@end
```

:::caption{for="success"}
**您可能更喜欢使用该标签**
:::

```edge
@flashMessage('notification')
  <div class="notification {{ $message.type }}">
    {{ $message.message }}
  </div>
@end
```

## @error

`@error` 标签为读取存储在 `flashMessages` 中 `errorsBag` 键内的错误消息提供了更好的 DX。

:::caption{for="error"}
**而不是编写条件语句**
:::

```edge
@if(flashMessages.has('errorsBag.E_BAD_CSRF_TOKEN'))
  <p>{{ flashMessages.get('errorsBag.E_BAD_CSRF_TOKEN') }}</p>
@end
```

:::caption{for="success"}
**您可能更喜欢使用该标签**
:::

```edge
@error('E_BAD_CSRF_TOKEN')
  <p>{{ $message }}</p>
@end
```

## @inputError

`@inputError` 标签为读取存储在 `flashMessages` 中 `inputErrorsBag` 键内的验证错误消息提供了更好的 DX。

:::caption{for="error"}
**而不是编写条件语句**
:::

```edge
@if(flashMessages.has('inputErrorsBag.title'))
  @each(message in flashMessages.get('inputErrorsBag.title'))
    <p>{{ message }}</p>
  @end
@end
```

:::caption{for="success"}
**您可能更喜欢使用该标签**
:::

```edge
@inputError('title')
  @each(message in $messages)
    <p>{{ message }}</p>
  @end
@end
```

## @vite

`@vite` 标签接受一个入口点路径数组，并返回相同的 `script` 和 `link` 标签。您提供给 `@vite` 标签的路径应与 `vite.config.js` 文件中注册的路径完全匹配。

```ts
export default defineConfig({
  plugins: [
    adonisjs({
      // highlight-start
      entrypoints: ['resources/js/app.js'],
      // highlight-end
    }),
  ],
})
```

```edge
@vite(['resources/js/app.js'])
```

您可以将脚本标签属性定义为第 2 个参数。例如：

```edge
@vite(['resources/js/app.js'], {
  defer: true,
})
```

## @viteReactRefresh

`@viteReactRefresh` 标签返回一个[脚本标签以启用 React HMR](https://vitejs.dev/guide/backend-integration.html#:~:text=you%27ll%20also%20need%20to%20add%20this%20before%20the%20above%20scripts)，用于使用 [@vitejs/plugin-react](https://www.npmjs.com/package/@vitejs/plugin-react) 包的项目。

```edge
@viteReactRefresh()
```

输出 HTML

```html
<script type="module">
  import RefreshRuntime from 'http://localhost:5173/@react-refresh'
  RefreshRuntime.injectIntoGlobalHook(window)
  window.$RefreshReg$ = () => {}
  window.$RefreshSig$ = () => (type) => type
  window.__vite_plugin_react_preamble_installed__ = true
</script>
```

## @can/@cannot

`@can` 和 `@cannot` 标签允许您通过引用能力名称或策略名称作为字符串在 Edge 模板中编写授权检查。

第一个参数是能力或策略引用，后跟检查接受的参数。

另请参阅: [预注册能力和策略](../security/authorization.md#pre-registering-abilities-and-policies)

```edge
@can('editPost', post)
  {{-- 可以编辑帖子 --}}
@end

@can('PostPolicy.edit', post)
  {{-- 可以编辑帖子 --}}
@end
```

```edge
@cannot('editPost', post)
  {{-- 无法编辑帖子 --}}
@end

@cannot('editPost', post)
  {{-- 无法编辑帖子 --}}
@end
```
