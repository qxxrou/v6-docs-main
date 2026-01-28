---
summary: 了解配置提供者以及它们如何帮助您在应用程序启动后延迟计算配置。
---

# Config providers

某些配置文件（如 `config/hash.ts`）不会导出普通对象形式的配置。相反，它们导出一个[配置提供者](https://github.com/adonisjs/core/blob/main/src/config_provider.ts#L16)。配置提供者为包提供了一个透明的API，允许它们在应用程序启动后延迟计算配置。

## 不使用配置 Config providers

要理解配置提供者，让我们看看如果不使用配置提供者，`config/hash.ts` 文件会是什么样子。

```ts
import { Scrypt } from '@adonisjs/core/hash/drivers/scrypt'

export default {
  default: 'scrypt',
  list: {
    scrypt: () =>
      new Scrypt({
        cost: 16384,
        blockSize: 8,
        parallelization: 1,
        maxMemory: 33554432,
      }),
  },
}
```

到目前为止还不错。我们不是从`drivers` 集合中引用 `scrypt` 驱动程序，而是直接导入它并使用工厂函数返回一个实例。

假设 `Scrypt` 驱动程序需要一个 Emitter 类的实例，以便每次哈希值时发出事件。

```ts
import { Scrypt } from '@adonisjs/core/hash/drivers/scrypt'
// insert-start
import emitter from '@adonisjs/core/services/emitter'
// insert-end

export default {
  default: 'scrypt',
  list: {
    scrypt: () =>
      new Scrypt(
        {
          cost: 16384,
          blockSize: 8,
          parallelization: 1,
          maxMemory: 33554432,
          // insert-start
        },
        emitter,
      ),
    // insert-end
  },
}
```

**🚨 上面的例子会失败** 因为 AdonisJS [容器服务](./container_services.md) 在应用程序启动之前是不可用的，而配置文件是在应用程序启动阶段之前导入的。

### 这是 AdonisJS 架构的问题吗？🤷🏻‍♂️

其实不是。我们不要使用容器服务，而是在配置文件中直接创建 Emitter 类的实例。

```ts
import { Scrypt } from '@adonisjs/core/hash/drivers/scrypt'
// delete-start
import emitter from '@adonisjs/core/services/emitter'
// delete-end
// insert-start
import { Emitter } from '@adonisjs/core/events'
// insert-end

// insert-start
const emitter = new Emitter()
// insert-end

export default {
  default: 'scrypt',
  list: {
    scrypt: () =>
      new Scrypt(
        {
          cost: 16384,
          blockSize: 8,
          parallelization: 1,
          maxMemory: 33554432,
        },
        emitter,
      ),
  },
}
```

现在我们有了一个新问题。我们为 `Scrypt` 驱动程序创建的 `emitter` 实例对我们来说并不是全局可用的，无法导入和监听驱动程序发出的事件。

因此，你可能想把 `Emitter` 类的构造移到单独的文件中，并导出它的实例。这样，你可以将 emitter 实例传递给驱动程序，并用它来监听事件。

```ts
// title: start/emitter.ts
import { Emitter } from '@adonisjs/core/events'
export const emitter = new Emitter()
```

```ts
import { Scrypt } from '@adonisjs/core/hash/drivers/scrypt'
// delete-start
import { Emitter } from '@adonisjs/core/events'
// delete-end
// insert-start
import { emitter } from '#start/emitter'
// insert-end

// delete-start
const emitter = new Emitter()
// delete-end

export default {
  default: 'scrypt',
  list: {
    scrypt: () =>
      new Scrypt(
        {
          cost: 16384,
          blockSize: 8,
          parallelization: 1,
          maxMemory: 33554432,
        },
        emitter,
      ),
  },
}
```

上面的代码可以正常工作。然而，您正在手动构建应用程序所需的依赖项。结果，您的应用程序将有很多样板代码来连接所有内容。

在 AdonisJS 中，我们努力编写最少的样板代码，并使用 IoC 容器查找依赖项。

## 使用 config provider

现在，让我们重写 `config/hash.ts` 文件并这次使用配置提供者。配置提供者是一个接受 [instance of the Application class](./application.md) 并使用容器解析其依赖项的函数。

```ts
// highlight-start
import { configProvider } from '@adonisjs/core'
// highlight-end
import { Scrypt } from '@adonisjs/core/hash/drivers/scrypt'

export default {
  default: 'scrypt',
  list: {
    // highlight-start
    scrypt: configProvider.create(async (app) => {
      const emitter = await app.container.make('emitter')

      return () =>
        new Scrypt(
          {
            cost: 16384,
            blockSize: 8,
            parallelization: 1,
            maxMemory: 33554432,
          },
          emitter,
        )
    }),
    // highlight-end
  },
}
```

一旦您使用了 [hash](../security/hashing.md) 服务，`scrypt` 驱动程序的配置提供者将会被执行以解析其依赖项。因此，我们不会尝试查找 `emitter` 直到我们在代码其他地方使用哈希服务。

由于配置提供者是异步的，您可能希望通过动态导入懒加载 `Scrypt` 驱动程序。

```ts
import { configProvider } from '@adonisjs/core'
// delete-start
import { Scrypt } from '@adonisjs/core/hash/drivers/scrypt'
// delete-end

export default {
  default: 'scrypt',
  list: {
    scrypt: configProvider.create(async (app) => {
      // insert-start
      const { Scrypt } = await import('@adonisjs/core/hash/drivers/scrypt')
      // insert-end
      const emitter = await app.container.make('emitter')

      return () =>
        new Scrypt(
          {
            cost: 16384,
            blockSize: 8,
            parallelization: 1,
            maxMemory: 33554432,
          },
          emitter,
        )
    }),
  },
}
```

## 如何访问已解析的配置?

您可以直接从服务中获取已解析的配置。例如，在哈希服务的情况下，您可以按如下方式获得已解析配置的引用。

```ts
import hash from '@adonisjs/core/services/hash'
console.log(hash.config)
```
