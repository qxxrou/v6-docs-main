---
summary: 了解如何在 Ace 命令中定义和处理命令参数。
---

# 命令参数

参数是指在命令名称之后提到的位置参数。由于参数是位置的，因此有必要按正确顺序传递它们。

您必须将命令参数定义为类属性，并使用 `args` 装饰器进行装饰。参数将按它们在类中定义的相同顺序被接受。

在以下示例中，我们使用 `@args.string` 装饰器来定义接受字符串值的参数。

```ts
import { BaseCommand, args, flags } from '@adonisjs/core/ace'

export default class GreetCommand extends BaseCommand {
  static commandName = 'greet'
  static description = '按名称问候用户'

  @args.string()
  declare name: string

  run() {
    console.log(this.name)
  }
}
```

要在同一参数名称下接受多个值，您可以使用 `@args.spread` 装饰器。请注意，扩展参数必须是最后一个。

```ts
import { BaseCommand, args, flags } from '@adonisjs/core/ace'

export default class GreetCommand extends BaseCommand {
  static commandName = 'greet'
  static description = '按名称问候用户'

  // highlight-start
  @args.spread()
  declare names: string[]
  // highlight-start

  run() {
    console.log(this.names)
  }
}
```

## 参数名称和描述

参数名称显示在帮助屏幕上。默认情况下，参数名称是类属性名称的短横线表示。但是，您也可以定义自定义值。

```ts
@args.string({
  argumentName: 'user-name'
})
declare name: string
```

参数描述显示在帮助屏幕上，可以使用 `description` 选项设置。

```ts
@args.string({
  argumentName: 'user-name',
  description: '用户的名称'
})
declare name: string
```

## 具有默认值的可选参数

默认情况下，所有参数都是必需的。但是，您可以通过将 `required` 选项设置为 `false` 来使它们成为可选的。可选参数必须在最后。

```ts
@args.string({
  description: '用户的名称',
  required: false,
})
declare name?: string
```

您可以使用 `default` 属性为可选参数设置默认值。

```ts
@args.string({
  description: '用户的名称',
  required: false,
  default: 'guest'
})
declare name: string
```

## 处理参数值

使用 `parse` 方法，您可以在将参数值定义为类属性之前对其进行处理。

```ts
@args.string({
  argumentName: 'user-name',
  description: '用户的名称',
  parse (value) {
    return value ? value.toUpperCase() : value
  }
})
declare name: string
```

## 访问所有参数

您可以使用 `this.parsed.args` 属性访问运行命令时提到的所有参数。

```ts
import { BaseCommand, args, flags } from '@adonisjs/core/ace'

export default class GreetCommand extends BaseCommand {
  static commandName = 'greet'
  static description = '按名称问候用户'

  @args.string()
  declare name: string

  run() {
    console.log(this.parsed.args)
  }
}
```
