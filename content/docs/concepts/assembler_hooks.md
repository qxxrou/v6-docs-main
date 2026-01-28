---
summary: Assembler hooks are a way of executing code at specific points in the assembler lifecycle.
---

# Assembler hooks

Assembler hooks 是一种在汇编器生命周期的特定阶段执行代码的方法。需要提醒的是，Assembler 是 AdonisJS 的一部分，它使您能够启动开发服务器、构建应用程序和运行测试。

这些钩子对于文件生成、代码编译或注入自定义构建步骤等任务非常有帮助。

例如，`@adonisjs/vite` 包使用 `onBuildStarting` 钩子来注入一个构建前端资源的步骤。因此，当您运行 `node ace build` 时，`@adonisjs/vite` 包会在其余构建过程之前先构建您的前端资源。这是一个很好的例子，展示了如何使用钩子来自定义构建流程。

## 添加钩子

`Assembler` 钩子在 `adonisrc.ts` 文件中的 hooks 键下定义：

```ts
import { defineConfig } from '@adonisjs/core/app'

export default defineConfig({
  hooks: {
    onBuildCompleted: [() => import('my-package/hooks/on_build_completed')],
    onBuildStarting: [() => import('my-package/hooks/on_build_starting')],
    onDevServerStarted: [
      () => import('my-package/hooks/on_dev_server_started'),
    ],
    onSourceFileChanged: [
      () => import('my-package/hooks/on_source_file_changed'),
    ],
  },
})
```

可以在装配生命周期的每个阶段定义多个钩子。每个钩子都是要执行的函数数组。

我们建议使用动态导入来加载钩子。这可以确保仅在需要时才加载钩子，而不是不必要的加载。如果您直接在 `adonisrc.ts` 文件中编写钩子代码，可能会减慢应用程序的启动速度。

## 创建钩子

钩子只是一个简单的函数。让我们以一个应该执行自定义构建任务的钩子为例。

```ts
// title: hooks/on_build_starting.ts
import type { AssemblerHookHandler } from '@adonisjs/core/types/app'

const buildHook: AssemblerHookHandler = async ({ logger }) => {
  logger.info('Generating some files...')

  await myCustomLogic()
}

export default buildHook
```

请注意，钩子必须默认导出。

一旦定义了此钩子，您只需将其添加到 `adonisrc.ts` 文件中，如下所示：

```ts
// title: adonisrc.ts
import { defineConfig } from '@adonisjs/core/app'

export default defineConfig({
  hooks: {
    onBuildStarting: [() => import('./hooks/on_build_starting')],
  },
})
```

现在，每次运行 `node ace build` 时，都会执行带有您定义的自定义逻辑的 `onBuildStarting` 钩子。

## 钩子列表

以下是可用钩子的列表：

### onBuildStarting

该钩子在构建开始前执行。对于文件生成或注入自定义构建步骤等任务非常有用。

### onBuildCompleted

该钩子在构建完成后执行。也可用于自定义构建过程。

### onDevServerStarted

该钩子在 Adonis 开发服务器启动后执行。

### onSourceFileChanged

每次修改源文件（由` tsconfig.json` 包含的文件）时都会执行此钩子。您的钩子将接收到被修改文件的路径作为参数。
