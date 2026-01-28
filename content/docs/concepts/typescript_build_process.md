---
summary: 了解 AdonisJS 中的 TypeScript 构建过程
---

# TypeScript 构建过程

用 TypeScript 编写的应用程序必须先编译成 JavaScript，然后才能在生产环境中运行。

编译 TypeScript 源文件可以使用许多不同的构建工具来完成。然而，在 AdonisJS 中，我们坚持最直截了当的方法，并使用以下经过时间考验的工具。

:::note

所有下面提到的工具都作为开发依赖项预装在官方入门套件中。

:::

- **[TSC](https://www.typescriptlang.org/docs/handbook/compiler-options.html)** 是 TypeScript 的官方编译器。我们使用 TSC 来执行类型检查并创建生产构建。

- **[TS Node Maintained](https://github.com/thetutlage/ts-node-maintained)** 是 TypeScript 的即时编译器。它允许你在不将 TypeScript 文件编译为 JavaScript 的情况下执行它们，并证明是开发过程中的绝佳工具。

- **[SWC](https://swc.rs/)** 是用 Rust 编写的 TypeScript 编译器。我们在开发过程中与 TS Node 一起使用它，以使 JIT 过程极快。

| 工具      | 用途                  | 类型检查 |
|-----------|-----------------------|----------|
| `TSC`     | 创建生产构建          | 是       |
| `TS Node` | 开发                  | 否       |
| `SWC`     | 开发                  | 否       |

## 无需编译即可执行 TypeScript 文件

你可以使用 `ts-node-maintained/register/esm` 钩子来执行 TypeScript 文件，而无需编译它们。例如，你可以通过运行以下命令来启动 HTTP 服务器：

```sh
node --import=ts-node-maintained/register/esm bin/server.js
```

- `--import`：import 标志允许你指定一个导出模块解析和加载自定义钩子的模块。有关更多信息，请参阅 [Node.js 自定义钩子文档](https://nodejs.org/docs/latest-v22.x/api/module.html#customization-hooks)。

- `ts-node-maintained/register/esm`：指向 `ts-node-maintained/register/esm` 脚本的路径，该脚本注册生命周期钩子以执行 TypeScript 源到 JavaScript 的即时编译。

- `bin/server.js`：AdonisJS HTTP 服务器入口点文件的路径。**另请参阅：[关于文件扩展名的说明](#关于文件扩展名的说明)**

你也可以对其他 TypeScript 文件重复此过程。例如：

```sh
// title: 运行测试
node --import=ts-node-maintained/register/esm bin/test.ts
```

```sh
// title: 运行 ace 命令
node --import=ts-node-maintained/register/esm bin/console.ts
```

```sh
// title: 运行其他 TypeScript 文件
node --import=ts-node-maintained/register/esm path/to/file.ts
```

### 关于文件扩展名的说明

你可能已经注意到我们到处都使用 `.js` 文件扩展名，即使磁盘上的文件是以 `.ts` 文件扩展名保存的。

这是因为，使用 ES 模块时，TypeScript 强制你在导入和运行脚本时使用 `.js` 扩展名。你可以在 [TypeScript 文档](https://www.typescriptlang.org/docs/handbook/modules/theory.html#typescript-imitates-the-hosts-module-resolution-but-with-types) 中了解此选择背后的理论。

:::tip

如果你使用的是 TypeScript 5.7 或更高版本，你可以使用 `.ts` 扩展名导入 TypeScript 文件。这是通过 [相对路径的路径重写](https://devblogs.microsoft.com/typescript/announcing-typescript-5-7-beta/#path-rewriting-for-relative-paths) 功能实现的。

由于某些运行时允许你"就地"运行 TypeScript 代码并要求 `.ts` 扩展名，你可能更喜欢为了未来的兼容性而已经使用 `.ts` 扩展名。

:::

## 运行开发服务器

我们建议直接使用 `serve` 命令，而不是直接运行 `bin/server.js` 文件，原因如下：

- 该命令包含文件监视器，并在文件更改时重新启动开发服务器。
- `serve` 命令检测你的应用程序使用的前端资源打包器并启动其开发服务器。例如，如果你的项目根目录中有 `vite.config.js` 文件，`serve` 命令将启动 `vite` 开发服务器。

```sh
node ace serve --watch
```

你可以使用 `--assets-args` 命令行标志将参数传递给 Vite 开发服务器。

```sh
node ace serve --watch --assets-args="--debug --base=/public"
```

你可以使用 `--no-assets` 标志来禁用 Vite 开发服务器。

```sh
node ace serve --watch --no-assets
```

### 向 Node.js 命令行传递选项

`serve` 命令将开发服务器（`bin/server.ts 文件`）作为子进程启动。如果你想将 [node 参数](https://nodejs.org/api/cli.html#options) 传递给子进程，你可以在命令名称之前定义它们。

```sh
node ace --no-warnings --inspect serve --watch
```

## 创建生产构建

AdonisJS 应用程序的生产构建是使用 `node ace build` 命令创建的。`build` 命令执行以下操作，在 `./build` 目录内创建[**独立的 JavaScript 应用程序**](#什么是独立构建)。

- 删除现有的 `./build` 文件夹（如果有）。
- **从头开始**重写 `ace.js` 文件以删除 `ts-node/esm` 加载器。
- 使用 Vite 编译前端资源（如果已配置）。
- 使用 [`tsc`](https://www.typescriptlang.org/docs/handbook/compiler-options.html) 将 TypeScript 源代码编译为 JavaScript。
- 将注册在 [`metaFiles`](../concepts/adonisrc_file.md#metafiles) 数组下的非 TypeScript 文件复制到 `./build` 文件夹。
- 将 `package.json` 和 `package-lock.json/yarn.lock` 文件复制到 `./build` 文件夹。

:::warning
对 `ace.js` 文件的任何修改都将在构建过程中丢失，因为该文件会从头开始重写。如果你想在 Ace 启动之前运行任何额外的代码，你应该改为在 `bin/console.ts` 文件中进行。
:::

就是这样！

```sh
node ace build
```

创建构建后，你可以 `cd` 进入 `build` 文件夹，安装生产依赖项，并运行你的应用程序。

```sh
cd build

# 安装生产依赖项
npm ci --omit=dev

# 运行服务器
node bin/server.js
```

你可以使用 `--assets-args` 命令行标志将参数传递给 Vite 构建命令。

```sh
node ace build --assets-args="--debug --base=/public"
```

你可以使用 `--no-assets` 标志来避免编译前端资源。

```sh
node ace build --no-assets
```

### 什么是独立构建？

独立构建指的是你的应用程序的 JavaScript 输出，你可以在没有原始 TypeScript 源的情况下运行它。

创建独立构建有助于减少你在生产服务器上部署的代码大小，因为你不必同时复制源文件和 JavaScript 输出。

创建生产构建后，你可以将 `./build` 复制到你的生产服务器，安装依赖项，定义环境变量，并运行应用程序。
