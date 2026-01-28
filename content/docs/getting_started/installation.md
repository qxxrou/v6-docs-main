---
summary: 如何创建和配置新的 AdonisJS 应用程序。
---

# 安装

在创建新应用程序之前，你应该确保你的计算机上已安装 Node.js 和 npm。**AdonisJS 需要 Node.js 版本 20 或更高版本**。

你可以使用[官方安装程序](https://nodejs.org/en/download/)或 [Volta](https://docs.volta.sh/guide/getting-started) 安装 Node.js。Volta 是一个跨平台包管理器，可在你的计算机上安装和运行多个 Node.js 版本。

```sh
// title: 验证 Node.js 版本
node -v
# v22.0.0
```

:::tip
**你更喜欢视觉学习吗？** - 查看我们朋友在 Adocasts 提供的 [Let's Learn AdonisJS 6](https://adocasts.com/series/lets-learn-adonisjs-6) 免费屏幕录制系列。
:::


## 创建新应用程序

你可以使用 [npm init](https://docs.npmjs.com/cli/v7/commands/npm-init) 创建新项目。这些命令将下载 [create-adonisjs](http://npmjs.com/create-adonisjs) 初始化程序包并开始安装过程。

你可以使用以下 CLI 标志之一自定义初始项目输出。

- `--kit`: 为项目选择[启动工具包](#starter-kits)。你可以在 **web**、**api**、**slim** 或 **inertia** 之间进行选择。

- `--db`: 指定你选择的数据库方言。你可以在 **sqlite**、**postgres**、**mysql** 或 **mssql** 之间进行选择。

- `--git-init`: 初始化 git 仓库。默认为 `false`。

- `--auth-guard`: 指定你选择的身份验证守卫。你可以在 **session**、**access_tokens** 或 **basic_auth** 之间进行选择。

:::codegroup

```sh
// title: npm
npm init adonisjs@latest hello-world
```

:::

在使用 `npm init` 命令传递 CLI 标志时，确保使用[双破折号两次](https://stackoverflow.com/questions/43046885/what-does-do-when-running-an-npm-command)。否则，`npm init` 不会将标志传递给 `create-adonisjs` 初始化程序包。例如：

```sh
# 创建项目并获取所有选项的提示
npm init adonisjs@latest hello-world

# 使用 MySQL 创建项目
npm init adonisjs@latest hello-world -- --db=mysql

# 使用 PostgreSQL 和 API 启动工具包创建项目
npm init adonisjs@latest hello-world -- --db=postgres --kit=api

# 使用 API 启动工具包和访问令牌守卫创建项目
npm init adonisjs@latest hello-world -- --kit=api --auth-guard=access_tokens
```

## 启动工具包

启动工具包用作使用 AdonisJS 创建应用程序的起点。它们带有[固执己见的文件夹结构](./folder_structure.md)、预配置的 AdonisJS 包以及开发期间所需的必要工具。


:::note

官方启动工具包使用 ES 模块和 TypeScript。这种组合允许你使用现代 JavaScript 结构并利用静态类型安全。

:::

### Web 启动工具包

Web 启动工具包专为创建传统的服务器渲染 Web 应用程序而定制。不要让**"传统"**这个词打击你。如果你的 Web 应用程序具有有限的前端交互性，我们推荐此启动工具包。

使用 [Edge.js](https://edgejs.dev) 在服务器上渲染 HTML 的简单性将提高你的生产力，因为你无需处理复杂的构建系统来渲染一些 HTML。

稍后，你可以使用 [Hotwire](https://hotwired.dev)、[HTMX](http://htmx.org) 或 [Unpoly](http://unpoly.com) 使你的应用程序像 SPA 一样导航，并使用 [Alpine.js](http://alpinejs.dev) 创建交互式小部件，如下拉菜单或模态框。

```sh
npm init adonisjs@latest -- -K=web

# 切换数据库方言
npm init adonisjs@latest -- -K=web --db=mysql
```

web 启动工具包附带以下包。

<table>
<thead>
<tr>
<th width="180px">包</th>
<th>描述</th>
</tr>
</thead>
<tbody><tr>
<td><code>@adonisjs/core</code></td>
<td>框架的核心具有你在创建后端应用程序时可能会用到的基线功能。</td>
</tr>
<tr>
<td><code>edge.js</code></td>
<td>用于编写 HTML 页面的 <a href="https://edgejs.dev">edge</a> 模板引擎。</td>
</tr>
<tr>
<td><code>@vinejs/vine</code></td>
<td><a href="https://vinejs.dev">VineJS</a> 是 Node.js 生态系统中最快的验证库之一。</td>
</tr>
<tr>
<td><code>@adonisjs/lucid</code></td>
<td>Lucid 是由 AdonisJS 核心团队维护的 SQL ORM。</td>
</tr>
<tr>
<td><code>@adonisjs/auth</code></td>
<td>框架的身份验证层。它配置为使用会话。</td>
</tr>
<tr>
<td><code>@adonisjs/shield</code></td>
<td>一组安全原语，可保护你的 Web 应用程序免受 <strong>CSRF</strong> 和 <strong>XSS</strong> 等攻击。</td>
</tr>
<tr>
<td><code>@adonisjs/static</code></td>
<td>用于从应用程序的 <code>/public</code> 目录提供静态资产的中介件。</td>
</tr>
<tr>
<td><code>vite</code></td>
<td><a href="https://vitejs.dev/">Vite</a> 用于编译前端资产。</td>
</tr>
</tbody></table>

---

### API 启动工具包

API 启动工具包专为创建 JSON API 服务器而定制。它是 `web` 启动工具包的精简版本。如果你计划使用 React 或 Vue 构建前端应用程序，你可以使用 API 启动工具包创建你的 AdonisJS 后端。

```sh
npm init adonisjs@latest -- -K=api

# 切换数据库方言
npm init adonisjs@latest -- -K=api --db=mysql
```

在此启动工具包中：

- 我们取消了对提供静态文件的支持。
- 不配置视图层和 vite。
- 关闭 XSS 和 CSRF 保护并启用 CORS 保护。
- 使用 ContentNegotiation 中介件以 JSON 格式发送 HTTP 响应。

API 启动工具包配置为基于会话的身份验证。然而，如果你希望使用基于令牌的身份验证，可以使用 `--auth-guard` 标志。

另请参阅：[我应该使用哪种身份验证守卫？](../authentication/introduction.md#choosing-an-auth-guard)

```sh
npm init adonisjs@latest -- -K=api --auth-guard=access_tokens
```

---

### Slim 启动工具包

对于极简主义者，我们创建了一个 `slim` 启动工具包。它仅带有框架的核心和默认的文件夹结构。当你不希望 AdonisJS 有任何花哨的功能时，可以使用它。

```sh
npm init adonisjs@latest -- -K=slim

# 切换数据库方言
npm init adonisjs@latest -- -K=slim --db=mysql
```

---

### Inertia 启动工具包

[Inertia](https://inertiajs.com/) 是一种构建服务器驱动的单页应用程序的方法。你可以使用你最喜欢的前端框架（React、Vue、Solid、Svelte）来构建应用程序的前端。

你可以使用 `--adapter` 标志选择要使用的框架。可用选项有 `react`、`vue`、`solid` 和 `svelte`。

你还可以使用 `--ssr` 和 `--no-ssr` 标志打开或关闭服务器端渲染。

```sh
npm init adonisjs@latest -- -K=inertia

# 使用服务器端渲染的 React
npm init adonisjs@latest -- -K=inertia --adapter=react --ssr

# 没有服务器端渲染的 Vue
npm init adonisjs@latest -- -K=inertia --adapter=vue --no-ssr
```

---

### 带上你的启动工具包

启动工具包是托管在 Git 仓库提供商（如 GitHub、Bitbucket 或 GitLab）上的预构建项目。你还可以创建自己的启动工具包，并按如下方式下载它们。

```sh
npm init adonisjs@latest -- -K="github_user/repo"

# 从 GitLab 下载
npm init adonisjs@latest -- -K="gitlab:user/repo"

# 从 Bitbucket 下载
npm init adonisjs@latest -- -K="bitbucket:user/repo"
```

你可以使用 Git+SSH 身份验证使用 `git` 模式下载私有仓库。

```sh
npm init adonisjs@latest -- -K="user/repo" --mode=git
```

最后，你可以指定标签、分支或提交。

```sh
# 分支
npm init adonisjs@latest -- -K="user/repo#develop"

# 标签
npm init adonisjs@latest -- -K="user/repo#v2.1.0"
```

## 启动开发服务器

创建 AdonisJS 应用程序后，你可以通过运行 `node ace serve` 命令来启动开发服务器。

Ace 是捆绑在框架核心内的命令行框架。`--hmr` 标志监视文件系统并对代码库的某些部分执行[热模块替换 (HMR)](../concepts/hmr.md)。

```sh
node ace serve --hmr
```

开发服务器运行后，你可以访问 [http://localhost:3333](http://localhost:3333) 在浏览器中查看你的应用程序。

## 为生产环境构建

由于 AdonisJS 应用程序是用 TypeScript 编写的，因此它们必须在生产环境中运行之前编译为 JavaScript。

你可以使用 `node ace build` 命令创建 JavaScript 输出。JavaScript 输出写入 `build` 目录。

配置 Vite 后，此命令还会使用 Vite 编译前端资产，并将输出写入 `build/public` 文件夹。

另请参阅：[TypeScript 构建过程](../concepts/typescript_build_process.md)。

```sh
node ace build
```

## 配置开发环境

虽然 AdonisJS 负责构建最终用户应用程序，但你可能需要额外的工具来享受开发过程并在编码风格上保持一致性。

我们强烈建议你使用 **[ESLint](https://eslint.org/)** 来检查你的代码，并使用 **[Prettier](https://prettier.io)** 重新格式化你的代码以保持一致性。

官方启动工具包附带 ESLint 和 Prettier，并使用来自 AdonisJS 核心团队的固执己见的预设。你可以在文档的[工具配置](../concepts/tooling_config.md)部分中了解有关它们的更多信息。

最后，我们建议你为代码编辑器安装 ESLint 和 Prettier 插件，以便在应用程序开发期间有更紧密的反馈循环。此外，你可以使用以下命令从命令行`检查`和`格式化`你的代码。

```sh
# 运行 ESLint
npm run lint

# 运行 ESLint 并自动修复问题
npm run lint -- --fix

# 运行 prettier
npm run format
```

## VSCode 扩展

你可以在支持 TypeScript 的任何代码编辑器上开发 AdonisJS 应用程序。然而，我们为 VSCode 开发了几个扩展，以进一步增强开发体验。

- [**AdonisJS**](https://marketplace.visualstudio.com/items?itemName=jripouteau.adonis-vscode-extension) - 查看应用程序路由、运行 ace 命令、迁移数据库以及直接从你的代码编辑器阅读文档。

- [**Edge**](https://marketplace.visualstudio.com/items?itemName=AdonisJS.vscode-edge) - 通过支持语法高亮、自动完成和代码片段，增强你的开发工作流程。

- [**Japa**](https://marketplace.visualstudio.com/items?itemName=jripouteau.japa-vscode) - 使用键盘快捷键或直接从活动侧边栏运行测试，无需离开你的代码编辑器即可运行测试。
