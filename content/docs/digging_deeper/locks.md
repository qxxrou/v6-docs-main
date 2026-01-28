---
summary: 使用`@adonisjs/lock`包管理AdonisJS应用中的原子锁。
---

# 原子锁

原子锁（也称为`mutex`）用于同步对共享资源的访问。换句话说，它防止多个进程或并发代码同时执行一段代码。

AdonisJS团队创建了一个名为[Verrou](https://github.com/Julien-R44/verrou)的与框架无关的包。`@adonisjs/lock`包基于此包，**因此请确保同时阅读[Verrou文档](https://verrou.dev/docs/introduction)，该文档更详细。**

## 安装

使用以下命令安装和配置该包：

```sh
node ace add @adonisjs/lock
```

:::disclosure{title="查看add命令执行的步骤"}

1. 使用检测到的包管理器安装`@adonisjs/lock`包。

2. 在`adonisrc.ts`文件中注册以下服务提供者：

   ```ts
   {
     providers: [
       // ...其他提供者
       () => import('@adonisjs/lock/lock_provider'),
     ]
   }
   ```

3. 创建`config/lock.ts`文件。

4. 在`start/env.ts`文件中定义以下环境变量及其验证：

   ```ts
   LOCK_STORE = redis
   ```

5. 如果使用`database`存储，可选地为`locks`表创建数据库迁移。

:::

## 配置

锁的配置存储在`config/lock.ts`文件中。

```ts
import env from '#start/env'
import { defineConfig, stores } from '@adonisjs/lock'

const lockConfig = defineConfig({
  default: env.get('LOCK_STORE'),
  stores: {
    redis: stores.redis({}),

    database: stores.database({
      tableName: 'locks',
    }),

    memory: stores.memory(),
  },
})

export default lockConfig

declare module '@adonisjs/lock/types' {
  export interface LockStoresList extends InferLockStores<typeof lockConfig> {}
}
```

<dl>

<dt>

default

</dt>

<dd>

用于管理锁的`default`存储。该存储在同一配置文件中的`stores`对象中定义。

</dd>

<dt>

stores

</dt>

<dd>

您计划在应用中使用的存储集合。我们建议始终配置`memory`存储，该存储可用于测试期间。

</dd>

</dl>

---

### 环境变量

默认锁存储使用`LOCK_STORE`环境变量定义，因此您可以在不同的环境中切换不同的存储。例如，在测试期间使用`memory`存储，在开发和生产环境中使用`redis`存储。

此外，环境变量必须经过验证，以允许预配置的存储之一。验证在`start/env.ts`文件中使用`Env.schema.enum`规则定义。

```ts
{
  LOCK_STORE: Env.schema.enum(['redis', 'database', 'memory'] as const),
}
```

### Redis存储

`redis`存储依赖于`@adonisjs/redis`包；因此，您必须在使用Redis存储之前配置此包。

以下是Redis存储接受的选项列表：

```ts
{
  redis: stores.redis({
    connectionName: 'main',
  }),
}
```

<dl>
<dt>
connectionName
</dt>
<dd>

`connectionName`属性引用`config/redis.ts`文件中定义的连接。

</dd>
</dl>

### 数据库存储

`database`存储依赖于`@adonisjs/lucid`包，因此您必须在使用数据库存储之前配置此包。

以下是数据库存储接受的选项列表：

```ts
{
  database: stores.database({
    connectionName: 'postgres',
    tableName: 'my_locks',
  }),
}
```

<dl>

<dt>

connectionName

</dt>

<dd>

引用`config/database.ts`文件中定义的数据库连接。如果未定义，我们将使用默认数据库连接。

</dd>

<dt>

tableName

</dt>

<dd>

用于存储速率限制的数据库表。

</dd>

</dl>

### 内存存储

`memory`存储是一个简单的内存存储，可用于测试目的，但不仅限于此。有时，对于某些用例，您可能希望拥有仅对当前进程有效且不跨多个进程共享的锁。

内存存储基于[`async-mutex`](https://www.npmjs.com/package/async-mutex)包构建。

```ts
{
  memory: stores.memory(),
}
```

## 锁定资源

一旦配置了锁存储，您就可以开始在应用中的任何位置使用锁来保护您的资源。

以下是如何使用锁保护资源的简单示例：

:::codegroup

```ts
// title: 手动锁定
import { errors } from '@adonisjs/lock'
import locks from '@adonisjs/lock/services/main'
import { HttpContext } from '@adonisjs/core/http'

export default class OrderController {
  async process({ response, request }: HttpContext) {
    const orderId = request.input('order_id')

    /**
     * 尝试立即获取锁（不重试）
     */
    const lock = locks.createLock(`order.processing.${orderId}`)
    const acquired = await lock.acquireImmediately()
    if (!acquired) {
      return 'Order is already being processed'
    }

    /**
     * 已获取锁。我们可以处理订单
     */
    try {
      await processOrder()
      return 'Order processed successfully'
    } finally {
      /**
       * 始终使用`finally`块释放锁，这样即使在处理过程中抛出异常，我们也能确保锁被释放
       */
      await lock.release()
    }
  }
}
```

```ts
// title: 自动锁定
import { errors } from '@adonisjs/lock'
import locks from '@adonisjs/lock/services/main'
import { HttpContext } from '@adonisjs/core/http'

export default class OrderController {
  async process({ response, request }: HttpContext) {
    const orderId = request.input('order_id')

    /**
     * 仅当锁可用时运行该函数
     * 锁也将在函数执行完毕后自动释放
     */
    const [executed, result] = await locks
      .createLock(`order.processing.${orderId}`)
      .runImmediately(async (lock) => {
        /**
         * 已获取锁。我们可以处理订单
         */
        await processOrder()
        return 'Order processed successfully'
      })

    /**
     * 无法获取锁且函数未执行
     */
    if (!executed) return 'Order is already being processed'

    return result
  }
}
```

:::

这是一个如何在应用中使用锁的快速示例。

它们还有许多其他方法可用于管理锁，例如`extend`用于延长锁的持续时间，`getRemainingTime`用于获取锁过期前的剩余时间，用于配置锁的选项等。

**为此，请确保阅读[Verrou文档](https://verrou.dev/docs/introduction)以获取更多详细信息**。提醒一下，`@adonisjs/lock`包基于`Verrou`包，因此您在Verrou文档中阅读的所有内容也适用于`@adonisjs/lock`包。

## 使用另一个存储

如果您在`config/lock.ts`文件中定义了多个存储，可以使用`use`方法为特定锁使用不同的存储。

```ts
import locks from '@adonisjs/lock/services/main'

const lock = locks.use('redis').createLock('order.processing.1')
```

否则，如果仅使用`default`存储，可以省略`use`方法。

```ts
import locks from '@adonisjs/lock/services/main'

const lock = locks.createLock('order.processing.1')
```

## 管理跨多个进程的锁

有时，您可能希望一个进程创建并获取锁，而另一个进程释放它。例如，您可能希望在Web请求中获取锁，并在后台作业中释放它。这可以使用`restoreLock`方法实现。

```ts
// title: 您的主服务器
import locks from '@adonisjs/lock/services/main'

export class OrderController {
  async process({ response, request }: HttpContext) {
    const orderId = request.input('order_id')

    const lock = locks.createLock(`order.processing.${orderId}`)
    await lock.acquire()

    /**
     * 调度一个后台作业来处理订单。
     *
     * 我们还将序列化的锁传递给作业，以便作业可以在订单处理完成后释放锁。
     */
    queue.dispatch('app/jobs/process_order', {
      lock: lock.serialize(),
    })
  }
}
```

```ts
// title: 您的后台作业
import locks from '@adonisjs/lock/services/main'

export class ProcessOrder {
  async handle({ lock }) {
    /**
     * 我们正在从序列化版本恢复锁
     */
    const handle = locks.restoreLock(lock)

    /**
     * 处理订单
     */
    await processOrder()

    /**
     * 释放锁
     */
    await handle.release()
  }
}
```

## 测试

在测试期间，您可以使用`memory`存储来避免为获取锁而进行真实的网络请求。您可以通过在`.env.testing`文件中将`LOCK_STORE`环境变量设置为`memory`来实现这一点。

```env
// title: .env.test
LOCK_STORE=memory
```

## 创建自定义锁存储

首先，请确保查看[Verrou文档](https://verrou.dev/docs/custom-lock-store)，该文档更深入地介绍了自定义锁存储的创建。在AdonisJS中，它几乎是相同的。

让我们创建一个简单的Noop存储，它什么都不做。首先，我们必须创建一个实现`LockStore`接口的类。

```ts
import type { LockStore } from '@adonisjs/lock/types'

class NoopStore implements LockStore {
  /**
   * 将锁保存在存储中。如果给定的键已被锁定，此方法应返回false
   *
   * @param key 要锁定的键
   * @param owner 锁的所有者
   * @param ttl 锁的生存时间（以毫秒为单位）。Null表示没有过期
   *
   * @returns 如果获取了锁，则返回true，否则返回false
   */
  async save(key: string, owner: string, ttl: number | null): Promise<boolean> {
    return false
  }

  /**
   * 如果锁由给定的所有者拥有，则从存储中删除锁。否则应抛出E_LOCK_NOT_OWNED错误
   *
   * @param key 要删除的键
   * @param owner 所有者
   */
  async delete(key: string, owner: string): Promise<void> {
    return false
  }

  /**
   * 强制从存储中删除锁，不检查所有者
   */
  async forceDelete(key: string): Promise<Void> {
    return false
  }

  /**
   * 检查锁是否存在。如果存在则返回true，否则返回false
   */
  async exists(key: string): Promise<boolean> {
    return false
  }

  /**
   * 延长锁过期时间。如果锁不属于给定的所有者，则抛出错误
   * 持续时间以毫秒为单位
   */
  async extend(key: string, owner: string, duration: number): Promise<void> {
    return false
  }
}
```

### 定义存储工厂

一旦创建了存储，您必须定义一个简单的工厂函数，该函数将由`@adonisjs/lock`用于创建存储的实例。

```ts
function noopStore(options: MyNoopStoreConfig) {
  return { driver: { factory: () => new NoopStore(options) } }
}
```

### 使用自定义存储

完成后，您可以按以下方式使用`noopStore`函数：

```ts
import { defineConfig } from '@adonisjs/lock'

const lockConfig = defineConfig({
  default: 'noop',
  stores: {
    noop: noopStore({}),
  },
})
```
