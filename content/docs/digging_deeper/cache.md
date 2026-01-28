---
summary: 缓存数据以提高应用程序的性能
---

# 缓存

AdonisJS Cache (`@adonisjs/cache`) 是一个基于 [bentocache.dev](https://bentocache.dev) 构建的简单、轻量级包装器，用于缓存数据并提高应用程序的性能。它提供了一个简单且统一的 API 来与各种缓存驱动程序交互，例如 Redis、DynamoDB、PostgreSQL、内存缓存等。

我们强烈建议您阅读 Bentocache 文档。Bentocache 提供了一些高级、可选的概念，在某些情况下非常有用，例如 [多层缓存](https://bentocache.dev/docs/multi-tier)、[宽限期](https://bentocache.dev/docs/grace-periods)、[标签](https://bentocache.dev/docs/tagging)、[超时](https://bentocache.dev/docs/timeouts)、[缓存击穿保护](https://bentocache.dev/docs/stampede-protection) 等。

## 安装

通过运行以下命令安装并配置 `@adonisjs/cache` 包：

```sh
node ace add @adonisjs/cache
```

:::disclosure{title="查看 add 命令执行的步骤"}

1. 使用检测到的包管理器安装 `@adonisjs/cache` 包。
2. 在 `adonisrc.ts` 文件中注册以下服务提供者：

   ```ts
   {
     providers: [
       // ...其他提供者
       () => import('@adonisjs/cache/cache_provider'),
     ]
   }
   ```

3. 创建 `config/cache.ts` 文件。
4. 在 `.env` 文件中定义所选缓存驱动程序的环境变量。

:::

## 配置

缓存包的配置文件位于 `config/cache.ts`。您可以配置默认缓存驱动程序、驱动程序列表及其特定配置。

另请参阅：[配置存根](https://github.com/adonisjs/cache/blob/1.x/stubs/config.stub)

```ts
import { defineConfig, store, drivers } from '@adonisjs/cache'

const cacheConfig = defineConfig({
  default: 'redis',

  stores: {
    /**
     * 仅在 DynamoDB 上缓存数据
     */
    dynamodb: store().useL2Layer(drivers.dynamodb({})),

    /**
     * 使用您配置的 Lucid 数据库缓存数据
     */
    database: store().useL2Layer(
      drivers.database({ connectionName: 'default' }),
    ),

    /**
     * 使用内存作为主存储，Redis 作为辅助存储来缓存数据。
     * 如果您的应用程序在多台服务器上运行，那么内存缓存
     * 需要使用总线进行同步。
     */
    redis: store()
      .useL1Layer(drivers.memory({ maxSize: '100mb' }))
      .useL2Layer(drivers.redis({ connectionName: 'main' }))
      .useBus(drivers.redisBus({ connectionName: 'main' })),
  },
})

export default cacheConfig
```

在上面的代码示例中，我们为每个缓存存储设置了多个层。这称为 [多层缓存系统](https://bentocache.dev/docs/multi-tier)。它让我们首先检查快速的内存缓存（第一层）。如果我们在那里找不到数据，我们将使用分布式缓存（第二层）。

### Redis

要将 Redis 用作您的缓存系统，您必须安装 `@adonisjs/redis` 包并配置它。请参阅此处的文档：[Redis](../database/redis.md)。

在 `config/cache.ts` 中，您必须指定 `connectionName`。此属性应与 `config/redis.ts` 文件中的 Redis 配置键匹配。

### 数据库

`database` 驱动程序与 `@adonisjs/lucid` 有对等依赖关系。因此，您必须安装并配置此包才能使用 `database` 驱动程序。

在 `config/cache.ts` 中，您必须指定 `connectionName`。此属性应与 `config/database.ts` 文件中的数据库配置键对应。

此外，配置 `database` 驱动程序时，[迁移](https://github.com/adonisjs/cache/blob/1.x/stubs/migration.stub) 将发布到您的 `database/migrations` 目录，您必须运行它才能创建存储缓存条目的必要表。

### 其他驱动程序

您可以使用其他驱动程序，例如 `memory`、`dynamodb`、`kysely` 和 `orchid`。

有关更多信息，请参阅 [缓存驱动程序](https://bentocache.dev/docs/cache-drivers)。

## 使用

配置好缓存后，您可以导入 `cache` 服务来与其交互。在以下示例中，我们将用户详细信息缓存 5 分钟：

```ts
import cache from '@adonisjs/cache/services/main'
import router from '@adonisjs/core/services/router'
import User from '#models/user'

router.get('/user/:id', async ({ params }) => {
  return cache.getOrSet({
    key: `user:${params.id}`,
    factory: async () => {
      const user = await User.find(params.id)
      return user.toJSON()
    },
    ttl: '5m',
  })
})
```

:::warning

如您所见，我们使用 `user.toJSON()` 序列化用户数据。这是必要的，因为您的数据必须序列化才能存储在缓存中。Lucid 模型或 `Date` 实例等类不能直接存储在 Redis 或数据库等缓存中。

:::

`ttl` 定义缓存键的生存时间。TTL 过期后，缓存键被视为过期，下一个请求将从 factory 方法重新获取数据。

### 标签

您可以将缓存条目与一个或多个标签关联，以简化失效。您可以将条目分组在多个标签下，而不是管理单个键，并在单个操作中使其失效。

```ts
await cache.getOrSet({
  key: 'foo',
  factory: getFromDb(),
  tags: ['tag-1', 'tag-2'],
})

await cache.deleteByTag({ tags: ['tag-1'] })
```

### 命名空间

另一种对键进行分组的方法是使用命名空间。这允许您稍后一次使所有内容失效：

```ts
const users = cache.namespace('users')

users.set({ key: '32', value: { name: 'foo' } })
users.set({ key: '33', value: { name: 'bar' } })

users.clear()
```

### 宽限期

您可以允许 Bentocache 返回过期数据（如果缓存键已过期但仍在宽限期内），使用 `grace` 选项。这使 Bentocache 的工作方式与 SWR 或 TanStack Query 等库相同。

```ts
import cache from '@adonisjs/cache/services/main'

cache.getOrSet({
  key: 'slow-api',
  factory: async () => {
    await sleep(5000)
    return 'slow-api-response'
  },
  ttl: '1h',
  grace: '6h',
})
```

在上面的示例中，数据将在 1 小时后被视为过期。但是，在 6 小时的宽限期内的下一个请求将返回过期数据，同时从 factory 方法重新获取数据并更新缓存。

### 超时

您可以使用 `timeout` 选项配置允许 factory 方法运行的最长时间，然后返回过期数据。默认情况下，Bentocache 设置 0ms 的软超时，这意味着我们总是返回过期数据，同时在后台重新获取数据。

```ts
import cache from '@adonisjs/cache/services/main'

cache.getOrSet({
  key: 'slow-api',
  factory: async () => {
    await sleep(5000)
    return 'slow-api-response'
  },
  ttl: '1h',
  grace: '6h',
  timeout: '200ms',
})
```

在上面的示例中，factory 方法最多允许运行 200ms。如果 factory 方法运行时间超过 200ms，将向用户返回过期数据，但 factory 方法将继续在后台运行。

如果您没有定义 `grace` 期，您仍然可以使用硬超时来在一定时间后停止 factory 方法。

```ts
import cache from '@adonisjs/cache/services/main'

cache.getOrSet({
  key: 'slow-api',
  factory: async () => {
    await sleep(5000)
    return 'slow-api-response'
  },
  ttl: '1h',
  hardTimeout: '200ms',
})
```

在这个例子中，factory 方法将在 200ms 后停止，并抛出错误。

:::note

您可以一起定义 `timeout` 和 `hardTimeout`。`timeout` 是 factory 方法在返回过期数据之前允许运行的最长时间，而 `hardTimeout` 是 factory 方法在停止之前允许运行的最长时间。

:::

## 缓存服务

从 `@adonisjs/cache/services/main` 导出的缓存服务是 [BentoCache](https://bentocache.dev/docs/named-caches) 类的单例实例，使用 `config/cache.ts` 中定义的配置创建。

您可以将缓存服务导入到您的应用程序中并使用它来与缓存交互：

```ts
import cache from '@adonisjs/cache/services/main'

/**
 * 如果不调用 `use` 方法，您在缓存服务上调用的方法
 * 将使用 `config/cache.ts` 中定义的默认存储。
 */
cache.put({ key: 'username', value: 'jul', ttl: '1h' })

/**
 * 使用 `use` 方法，您可以切换到 `config/cache.ts` 中定义的不同存储。
 */
cache.use('dynamodb').put({ key: 'username', value: 'jul', ttl: '1h' })
```

您可以在此处找到所有可用方法：[BentoCache API](https://bentocache.dev/docs/methods)。

```ts
await cache.namespace('users').set({ key: 'username', value: 'jul' })
await cache.namespace('users').get({ key: 'username' })

await cache.get({ key: 'username' })

await cache.set({ key: 'username', value: 'jul' })
await cache.setForever({ key: 'username', value: 'jul' })

await cache.getOrSet({
  key: 'username',
  factory: async () => fetchUserName(),
  ttl: '1h',
})

await cache.has({ key: 'username' })
await cache.missing({ key: 'username' })

await cache.pull({ key: 'username' })

await cache.delete({ key: 'username' })
await cache.deleteMany({ keys: ['products', 'users'] })
await cache.deleteByTag({ tags: ['products', 'users'] })

await cache.clear()
```

## Edge 助手

`cache` 服务作为 Edge 助手在您的视图中可用。您可以使用它直接在模板中检索缓存值。

```edge
<p>
  你好 {{ await cache.get('username') }}
</p>
```

## Ace 命令

`@adonisjs/cache` 包还提供了一组 Ace 命令，用于从终端与缓存交互。

### `cache:clear`

清除指定存储的缓存。如果未指定，它将清除默认存储。

```sh
# 清除默认缓存存储
node ace cache:clear

# 清除特定的缓存存储
node ace cache:clear redis

# 清除特定命名空间
node ace cache:clear store --namespace users

# 清除多个特定标签
node ace cache:clear store --tags products --tags users
```

### `cache:delete`

从指定存储中删除特定的缓存键。如果未指定，它将从默认存储中删除。

```sh
# 删除特定的缓存键
node ace cache:delete cache-key

# 从特定存储中删除特定的缓存键
node ace cache:delete cache-key store
```

### `cache:prune`

一些缓存驱动程序（如数据库驱动程序）不会自动删除过期的键，因为它们缺乏原生 TTL 支持。您可以使用 `cache:prune` 命令手动删除过期的键。在支持 TTL 的存储上，此命令将导致无操作。

```sh
# 从默认缓存存储中删除过期的键
node ace cache:prune

# 从特定缓存存储中删除过期的键
node ace cache:prune store
```
