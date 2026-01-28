---
summary: 学习如何使用AdonisJS日志记录器将日志写入控制台、文件和外部服务。该日志记录器基于Pino构建，速度快且支持多个目标。
---

# 日志记录器

AdonisJS内置了一个日志记录器，支持将日志写入**文件**、**标准输出**和**外部日志服务**。在底层，我们使用[pino](https://getpino.io/#/)。Pino是Node.js生态系统中最快的日志库之一，它生成[NDJSON格式](https://github.com/ndjson/ndjson-spec)的日志。

## 使用方法

首先，您可以从应用内的任何位置导入Logger服务来写入日志。日志将写入`stdout`并显示在终端上。

```ts
import logger from '@adonisjs/core/services/logger'

logger.info('this is an info message')
logger.error({ err: error }, 'Something went wrong')
```

建议在HTTP请求期间使用`ctx.logger`属性。HTTP上下文包含请求感知日志记录器的实例，该实例将当前请求ID添加到每个日志语句中。

```ts
import router from '@adonisjs/core/services/router'
import User from '#models/user'

router.get('/users/:id', async ({ logger, params }) => {
  logger.info('Fetching user by id %s', params.id)
  const user = await User.find(params.id)
})
```

## 配置

日志记录器的配置存储在`config/logger.ts`文件中。默认情况下，只配置了一个日志记录器。但是，如果您想在应用中使用多个日志记录器，可以定义多个日志记录器的配置。

```ts
// title: config/logger.ts
import env from '#start/env'
import { defineConfig } from '@adonisjs/core/logger'

export default defineConfig({
  default: 'app',

  loggers: {
    app: {
      enabled: true,
      name: env.get('APP_NAME'),
      level: env.get('LOG_LEVEL', 'info'),
    },
  },
})
```

<dl>
<dt>

default

<dt>

<dd>

`default`属性是对同一文件中`loggers`对象内配置的日志记录器之一的引用。

除非您在使用日志记录器API时选择特定的日志记录器，否则默认日志记录器将用于写入日志。

</dd>

<dt>

loggers

<dt>

<dd>

`loggers`对象是一个键值对，用于配置多个日志记录器。键是日志记录器的名称，值是[pino](https://getpino.io/#/docs/api?id=options)接受的配置对象。

</dd>
</dl>

## 传输目标

Pino中的传输在将日志写入目标时起着重要作用。您可以在配置文件中配置[多个目标](https://getpino.io/#/docs/api?id=transport-object)，Pino将把日志传递给所有目标。每个目标还可以指定它希望接收日志的级别。

:::note

如果您未在目标配置中定义`level`，则配置的目标将从父日志记录器继承该级别。

这种行为与pino不同。在Pino中，目标不会从父日志记录器继承级别。

:::

```ts
{
  loggers: {
    app: {
      enabled: true,
      name: env.get('APP_NAME'),
      level: env.get('LOG_LEVEL', 'info'),

      // highlight-start
      transport: {
        targets: [
          {
            target: 'pino/file',
            level: 'info',
            options: {
              destination: 1
            }
          },
          {
            target: 'pino-pretty',
            level: 'info',
            options: {}
          },
        ]
      }
      // highlight-end
    }
  }
}
```

<dl>
<dt>

File target

<dt>

<dd>

`pino/file`目标将日志写入文件描述符。`destination = 1`表示将日志写入`stdout`（这是文件描述符的标准[unix约定](https://en.wikipedia.org/wiki/File_descriptor)）。

</dd>

<dt>

Pretty target

<dt>

<dd>

`pino-pretty`目标使用[pino-pretty npm模块](http://npmjs.com/package/pino-pretty)将日志美化打印到文件描述符。

</dd>
</dl>

## 条件定义目标

根据代码运行的环境注册目标是很常见的。例如，在开发中使用`pino-pretty`目标，在生产中使用`pino/file`目标。

如下面所示，使用条件构造`targets`数组会使配置文件看起来不整洁。

```ts
import app from '@adonisjs/core/services/app'

loggers: {
  app: {
    transport: {
      targets: [
        ...(!app.inProduction
          ? [{ target: 'pino-pretty', level: 'info' }]
          : []),
        ...(app.inProduction ? [{ target: 'pino/file', level: 'info' }] : []),
      ]
    }
  }
}
```

因此，您可以使用`targets`辅助函数使用流畅的API定义条件数组项。我们在下面的示例中使用`targets.pushIf`方法表达相同的条件。

```ts
import { targets, defineConfig } from '@adonisjs/core/logger'

loggers: {
  app: {
    transport: {
      targets: targets()
        .pushIf(!app.inProduction, { target: 'pino-pretty', level: 'info' })
        .pushIf(app.inProduction, { target: 'pino/file', level: 'info' })
        .toArray()
    }
  }
}
```

为了进一步简化代码，您可以使用`targets.pretty`和`targets.file`方法为`pino/file`和`pino-pretty`目标定义配置对象。

```ts
import { targets, defineConfig } from '@adonisjs/core/logger'

loggers: {
  app: {
    transport: {
      targets: targets()
        .pushIf(app.inDev, targets.pretty())
        .pushIf(app.inProduction, targets.file())
        .toArray()
    }
  }
}
```

## 使用多个日志记录器

AdonisJS对配置多个日志记录器有一流的支持。日志记录器的唯一名称和配置在`config/logger.ts`文件中定义。

```ts
export default defineConfig({
  default: 'app',

  loggers: {
    // highlight-start
    app: {
      enabled: true,
      name: env.get('APP_NAME'),
      level: env.get('LOG_LEVEL', 'info'),
    },
    payments: {
      enabled: true,
      name: 'payments',
      level: env.get('LOG_LEVEL', 'info'),
    },
    // highlight-start
  },
})
```

配置完成后，您可以使用`logger.use`方法访问命名的日志记录器。

```ts
import logger from '@adonisjs/core/services/logger'

logger.use('payments')
logger.use('app')

// 获取默认日志记录器的实例
logger.use()
```

## 依赖注入

当使用依赖注入时，您可以将`Logger`类作为依赖项进行类型提示，IoC容器将解析在配置文件中定义的默认日志记录器的实例。

如果类是在HTTP请求期间构建的，则容器将注入日志记录器的请求感知实例。

```ts
import { inject } from '@adonisjs/core'
import { Logger } from '@adonisjs/core/logger'

// highlight-start
@inject()
// highlight-end
class UserService {
  // highlight-start
  constructor(protected logger: Logger) {}
  // highlight-end

  async find(userId: string | number) {
    this.logger.info('Fetching user by id %s', userId)
    const user = await User.find(userId)
  }
}
```

## 日志方法

Logger API几乎与Pino相同，除了AdonisJS日志记录器不是Event发射器的实例（而Pino是）。除此之外，日志方法与pino具有相同的API。

```ts
import logger from '@adonisjs/core/services/logger'

logger.trace(config, 'using config')
logger.debug('user details: %o', { username: 'virk' })
logger.info('hello %s', 'world')
logger.warn('Unable to connect to database')
logger.error({ err: Error }, 'Something went wrong')
logger.fatal({ err: Error }, 'Something went wrong')
```

可以将额外的合并对象作为第一个参数传递。然后，对象属性将添加到输出JSON中。

```ts
logger.info({ user: user }, 'Fetched user by id %s', user.id)
```

要显示错误，您可以[使用`err`键](https://getpino.io/#/docs/api?id=serializers-object)指定错误值。

```ts
logger.error({ err: error }, 'Unable to lookup user')
```

## 条件日志记录

日志记录器会为等于或高于配置文件中配置的级别的日志生成日志。例如，如果级别设置为`warn`，则会忽略`info`、`debug`和`trace`级别的日志。

如果为日志消息计算数据是昂贵的，您应该在计算数据之前检查给定的日志级别是否已启用。

```ts
import logger from '@adonisjs/core/services/logger'

if (logger.isLevelEnabled('debug')) {
  const data = await getLogData()
  logger.debug(data, 'Debug message')
}
```

您可以使用`ifLevelEnabled`方法表达相同的条件。该方法接受一个回调作为第二个参数，当指定的日志级别启用时，该回调将被执行。

```ts
logger.ifLevelEnabled('debug', async () => {
  const data = await getLogData()
  logger.debug(data, 'Debug message')
})
```

## 子日志记录器

子日志记录器是一个独立的实例，它继承父日志记录器的配置和绑定。

可以使用`logger.child`方法创建子日志记录器的实例。该方法接受绑定作为第一个参数，可选的配置对象作为第二个参数。

```ts
import logger from '@adonisjs/core/services/logger'

const requestLogger = logger.child({ requestId: ctx.request.id() })
```

子日志记录器也可以在不同的日志级别下记录日志。

```ts
logger.child({}, { level: 'warn' })
```

## Pino静态属性

[Pino静态](https://getpino.io/#/docs/api?id=statics)方法和属性由`@adonisjs/core/logger`模块导出。

```ts
import {
  multistream,
  destination,
  transport,
  stdSerializers,
  stdTimeFunctions,
  symbols,
  pinoVersion,
} from '@adonisjs/core/logger'
```

## 将日志写入文件

Pino附带了一个`pino/file`目标，您可以使用它将日志写入文件。在目标选项中，您可以指定日志文件目标路径。

```ts
app: {
  enabled: true,
  name: env.get('APP_NAME'),
  level: env.get('LOG_LEVEL', 'info')

  transport: {
    targets: targets()
      .push({
         transport: 'pino/file',
         level: 'info',
         options: {
           destination: '/var/log/apps/adonisjs.log'
         }
      })
      .toArray()
  }
}
```

### 文件轮换

Pino没有内置的文件轮换支持，因此您要么必须使用系统级工具，如[logrotate](https://getpino.io/#/docs/help?id=rotate)，要么使用第三方包，如[pino-roll](https://github.com/feugy/pino-roll)。

```sh
npm i pino-roll
```

```ts
app: {
  enabled: true,
  name: env.get('APP_NAME'),
  level: env.get('LOG_LEVEL', 'info')

  transport: {
    targets: targets()
      // highlight-start
      .push({
        target: 'pino-roll',
        level: 'info',
        options: {
          file: '/var/log/apps/adonisjs.log',
          frequency: 'daily',
          mkdir: true
        }
      })
      // highlight-end
     .toArray()
  }
}
```

## 隐藏敏感值

日志可能成为泄露敏感数据的来源。因此，建议观察您的日志并从输出中删除/隐藏敏感值。

在Pino中，您可以使用`redact`选项来隐藏/删除日志中的敏感键值对。在底层，使用[fast-redact](https://github.com/davidmarkclements/fast-redact)包，您可以查阅其文档以查看可用的表达式。

```ts
// title: config/logger.ts
app: {
  enabled: true,
  name: env.get('APP_NAME'),
  level: env.get('LOG_LEVEL', 'info')

  // highlight-start
  redact: {
    paths: ['password', '*.password']
  }
  // highlight-end
}
```

```ts
import logger from '@adonisjs/core/services/logger'

const username = request.input('username')
const password = request.input('password')

logger.info({ username, password }, 'user signup')
// output: {"username":"virk","password":"[Redacted]","msg":"user signup"}
```

默认情况下，该值将替换为`[Redacted]`占位符。您可以定义自定义占位符或删除键值对。

```ts
redact: {
  paths: ['password', '*.password'],
  censor: '[PRIVATE]'
}

// 删除属性
redact: {
  paths: ['password', '*.password'],
  remove: true
}
```

### 使用Secret数据类型

另一种替代方法是将敏感值包装在Secret类中。例如：

另见：[Secret类使用文档](../references/helpers.md#secret)

```ts
import { Secret } from '@adonisjs/core/helpers'

const username = request.input('username')
// delete-start
const password = request.input('password')
// delete-end
// insert-start
const password = new Secret(request.input('password'))
// insert-end

logger.info({ username, password }, 'user signup')
// output: {"username":"virk","password":"[redacted]","msg":"user signup"}
```
