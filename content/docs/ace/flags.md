---
summary: 学习如何在 Ace 命令中定义和处理命令标志。
---

# 命令标志

标志是使用两个连字符（`--`）或单个连字符（`-`）（称为标志别名）提到的命名参数。标志可以按任何顺序提到。

您必须将标志定义为类属性，并使用 `@flags` 装饰器进行装饰。在以下示例中，我们定义了 `resource` 和 `singular` 标志，两者都表示布尔值。

```ts
import { BaseCommand, flags } from '@adonisjs/core/ace'

export default class MakeControllerCommand extends BaseCommands {
  @flags.boolean()
  declare resource: boolean

  @flags.boolean()
  declare singular: boolean
}
```

## 标志类型

Ace 允许为以下类型之一定义标志。

### 布尔标志

布尔标志使用 `@flags.boolean` 装饰器定义。提到标志会将其值设置为 `true`。否则，标志值为 `undefined`。

```sh
make:controller --resource

# this.resource === true
```

```sh
make:controller

# this.resource === undefined
```

```sh
make:controller --no-resource

# this.resource === false
```

最后一个示例显示布尔标志可以用 `--no-` 前缀否定。

默认情况下，否定选项不会显示在帮助输出中。但是，您可以使用 `showNegatedVariantInHelp` 选项启用它。

```ts
export default class MakeControllerCommand extends BaseCommands {
  @flags.boolean({
    showNegatedVariantInHelp: true,
  })
  declare resource: boolean
}
```

### 字符串标志

字符串标志在标志名称后接受一个值。您可以使用 `@flags.string` 方法定义字符串标志。

```ts
import { BaseCommand, flags } from '@adonisjs/core/ace'

export default class MakeControllerCommand extends BaseCommands {
  @flags.string()
  declare model: string
}
```

```sh
make:controller --model user

# this.model = 'user'
```

如果标志值包含空格或特殊字符，则必须将其包装在引号 `""` 内。

```sh
make:controller --model blog user
# this.model = 'blog'

make:controller --model "blog user"
# this.model = 'blog user'
```

如果提到了标志但未提供值（即使标志是可选的），也会显示错误。

```sh
make:controller
# 可以工作！未提到可选标志

make:controller --model
# 错误！缺少值
```

### 数字标志

数字标志的解析与字符串标志类似。但是，该值会经过验证以确保它是有效数字。

您可以使用 `@flags.number` 装饰器创建数字标志。

```ts
import { BaseCommand, flags } from '@adonisjs/core/ace'

export default class MakeUserCommand extends BaseCommands {
  @flags.number()
  declare score: number
}
```

### 数组标志

数组标志允许在运行命令时多次使用该标志。您可以使用 `@flags.array` 方法定义数组标志。

```ts
import { BaseCommand, flags } from '@adonisjs/core/ace'

export default class MakeUserCommand extends BaseCommands {
  @flags.array()
  declare groups: string[]
}
```

```sh
make:user --groups=admin --groups=moderators --groups=creators

# this.groups = ['admin', 'moderators', 'creators']
```

## 标志名称和描述

默认情况下，标志名称是类属性名称的短横线表示。但是，您可以通过 `flagName` 选项定义自定义名称。

```ts
@flags.boolean({
  flagName: 'server'
})
declare startServer: boolean
```

标志描述显示在帮助屏幕上。您可以使用 `description` 选项定义它。

```ts
@flags.boolean({
  flagName: 'server',
  description: '启动应用程序服务器'
})
declare startServer: boolean
```

## 标志别名

别名是标志的简写名称，使用单个连字符（`-`）提到。别名必须是单个字符。

```ts
@flags.boolean({
  alias: ['r']
})
declare resource: boolean

@flags.boolean({
  alias: ['s']
})
declare singular: boolean
```

标志别名可以在运行命令时组合。

```ts
make:controller -rs

# 等同于
make:controller --resource --singular
```

## 默认值

您可以使用 `default` 选项为标志定义默认值。当未提到标志或提到标志但没有值时，使用默认值。

```ts
@flags.boolean({
  default: true,
})
declare startServer: boolean

@flags.string({
  default: 'sqlite',
})
declare connection: string
```

## 处理标志值

使用 `parse` 方法，您可以在将标志值定义为类属性之前对其进行处理。

```ts
@flags.string({
  parse (value) {
    return value ? connections[value] : value
  }
})
declare connection: string
```

## 访问所有标志

您可以使用 `this.parsed.flags` 属性访问运行命令时提到的所有标志。flags 属性是键值对对象。

```ts
import { BaseCommand, flags } from '@adonisjs/core/ace'

export default class MakeControllerCommand extends BaseCommands {
  @flags.boolean()
  declare resource: boolean

  @flags.boolean()
  declare singular: boolean

  async run() {
    console.log(this.parsed.flags)

    /**
     * 提到但未被命令接受的标志名称
     */
    console.log(this.parsed.unknownFlags)
  }
}
```
