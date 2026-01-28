---
summary: 了解 AdonisJS 用于 TypeScript、Prettier 和 ESLint 的工具配置预设。
---

# 工具配置

AdonisJS 高度依赖 TypeScript、Prettier 和 ESLint 来保持代码一致性、在构建时检查错误，更重要的是提供愉悦的开发体验。

我们已经将所有选择抽象到现成的配置预设中，供所有官方包和官方 starter kit 使用。

如果您想在使用 TypeScript 编写的 Node.js 应用中使用相同的配置预设，请继续阅读本指南。

## TSConfig

[`@adonisjs/tsconfig`](https://github.com/adonisjs/tooling-config/tree/main/packages/typescript-config) 包包含 TypeScript 项目的基础配置。我们将 TypeScript 模块系统设置为 `NodeNext`，并使用 `TS Node + SWC` 进行即时编译。

您可以随时浏览 [基础配置文件](https://github.com/adonisjs/tooling-config/blob/main/packages/typescript-config/tsconfig.base.json)、[应用配置文件](https://github.com/adonisjs/tooling-config/blob/main/packages/typescript-config/tsconfig.app.json) 和 [包开发配置文件](https://github.com/adonisjs/tooling-config/blob/main/packages/typescript-config/tsconfig.package.json) 中的选项。

您可以按照以下方式安装并使用该包。

```sh
npm i -D @adonisjs/tsconfig

# Make sure also to install the following packages
npm i -D typescript ts-node-maintained @swc/core
```

创建 AdonisJS 应用时，从 `tsconfig.app.json` 文件扩展。（Starter kit 已预先配置）

```jsonc
{
  "extends": "@adonisjs/tsconfig/tsconfig.app.json",
  "compilerOptions": {
    "rootDir": "./",
    "outDir": "./build",
  },
}
```

为 AdonisJS 生态系统创建包时，从 `tsconfig.package.json` 文件扩展。

```jsonc
{
  "extends": "@adonisjs/tsconfig/tsconfig.package.json",
  "compilerOptions": {
    "rootDir": "./",
    "outDir": "./build",
  },
}
```

## Prettier 配置

[`@adonisjs/prettier-config`](https://github.com/adonisjs/tooling-config/tree/main/packages/prettier-config) 包包含用于自动格式化源代码以保持一致样式的基础配置。您可以随时浏览 [index.json 文件](https://github.com/adonisjs/tooling-config/blob/main/packages/prettier-config/index.json) 中的配置选项。

您可以按照以下方式安装并使用该包。

```sh
npm i -D @adonisjs/prettier-config

# Make sure also to install prettier
npm i -D prettier
```

在 `package.json` 文件中定义以下属性。

```jsonc
{
  "prettier": "@adonisjs/prettier-config",
}
```

此外，创建一个 `.prettierignore` 文件以忽略特定文件和目录。

```
// title: .prettierignore
build
node_modules
```

## ESLint 配置

[`@adonisjs/eslint-config`](https://github.com/adonisjs/tooling-config/tree/main/packages/eslint-config) 包包含用于应用 linting 规则的基础配置。您可以随时浏览 [基础配置文件](https://github.com/adonisjs/tooling-config/blob/main/packages/eslint-config/presets/ts_base.js)、[应用配置文件](https://github.com/adonisjs/tooling-config/blob/main/packages/eslint-config/presets/ts_app.js) 和 [包开发配置文件](https://github.com/adonisjs/tooling-config/blob/main/packages/eslint-config/presets/ts_package.js) 中的选项。

您可以按照以下方式安装并使用该包。

:::note

我们的配置预设使用 [eslint-plugin-prettier](https://github.com/prettier/eslint-plugin-prettier) 来确保 ESLint 和 Prettier 可以协同工作而不会相互干扰。

:::

```sh
npm i -D @adonisjs/eslint-config

# Make sure also to install eslint
npm i -D eslint
```

创建 AdonisJS 应用时，从 `eslint-config/app` 文件扩展。（Starter kit 已预先配置）

```json
// title: package.json
{
  "eslintConfig": {
    "extends": "@adonisjs/eslint-config/app"
  }
}
```

为 AdonisJS 生态系统创建包时，从 `eslint-config/package` 文件扩展。

```json
// title: package.json
{
  "eslintConfig": {
    "extends": "@adonisjs/eslint-config/package"
  }
}
```

```

```
