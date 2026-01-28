---
summary: Lucid ORM 快速概览，一个基于 Knex 构建的 SQL 查询构建器和 Active Record ORM。
---

# Lucid ORM

Lucid 是一个 SQL 查询构建器，也是一个基于 [Knex](https://knexjs.org) 构建的 Active Record ORM，由 AdonisJS 核心团队创建和维护。Lucid 致力于充分发挥 SQL 的潜力，并为许多高级 SQL 操作提供简洁的 API。

:::note
Lucid 的文档可在 [https://lucid.adonisjs.com](https://lucid.adonisjs.com) 获取
:::

## 为什么选择 Lucid

以下是一些精选的 Lucid 特性：

- 基于 Knex 构建的流畅查询构建器。
- 支持读写副本和多连接管理。
- 遵循 Active Record 模式的基于类的模型（处理关系、序列化和钩子）。
- 使用增量变更集修改数据库架构的迁移系统。
- 用于生成测试假数据的模型工厂。
- 用于向数据库插入初始/虚拟数据的数据库填充器。

除此之外，以下是在 AdonisJS 应用程序中使用 Lucid 的其他原因：

- 我们为 Lucid 与 Auth 包和验证器提供了一流的集成。因此，您无需自己编写这些集成。

- Lucid 预配置了 `api` 和 `web` 入门套件，为您的应用程序提供了一个良好的开端。

- Lucid 的主要目标之一是充分发挥 SQL 的潜力，支持许多高级 SQL 操作，如**窗口函数**、**递归 CTE**、**JSON 操作**、**基于行的锁**等等。

- Lucid 和 Knex 都已经存在多年。因此，与许多其他新的 ORM 相比，它们更加成熟且经过实战检验。

话虽如此，AdonisJS 并不强制您使用 Lucid。只需卸载该软件包并安装您选择的 ORM 即可。

## 安装

使用以下命令安装和配置 Lucid。

```sh
node ace add @adonisjs/lucid
```

:::disclosure{title="查看配置命令执行的步骤"}

1. 在 `adonisrc.ts` 文件中注册以下服务提供者。

   ```ts
   {
     providers: [
       // ...其他提供者
       () => import('@adonisjs/lucid/database_provider'),
     ]
   }
   ```

2. 在 `adonisrc.ts` 文件中注册以下命令。

   ```ts
   {
     commands: [
       // ...其他命令
       () => import('@adonisjs/lucid/commands'),
     ]
   }
   ```

3. 创建 `config/database.ts` 文件。

4. 为所选方言定义环境变量及其验证。

5. 安装所需的对等依赖项。

:::

## 创建您的第一个模型

配置完成后，您可以使用以下命令创建您的第一个模型。

```sh
node ace make:model User
```

此命令在 `app/models` 目录中创建一个具有以下内容的新文件。

```ts
import { DateTime } from 'luxon'
import { BaseModel, column } from '@adonisjs/lucid/orm'

export default class User extends BaseModel {
  @column({ isPrimary: true })
  declare id: number

  @column.dateTime({ autoCreate: true })
  declare createdAt: DateTime

  @column.dateTime({ autoCreate: true, autoUpdate: true })
  declare updatedAt: DateTime
}
```

通过访问[官方文档](https://lucid.adonisjs.com/docs/models)了解更多关于模型的信息。

## 迁移

迁移是一种使用增量变更集修改数据库架构和数据的方法。您可以使用以下命令创建新的迁移。

```sh
node ace make:migration users
```

此命令在 `database/migrations` 目录中创建一个具有以下内容的新文件。

```ts
import { BaseSchema } from '@adonisjs/lucid/schema'

export default class extends BaseSchema {
  protected tableName = 'users'

  async up() {
    this.schema.createTable(this.tableName, (table) => {
      table.increments('id')
      table.timestamp('created_at')
      table.timestamp('updated_at')
    })
  }

  async down() {
    this.schema.dropTable(this.tableName)
  }
}
```

您可以使用以下命令运行所有待处理的迁移。

```sh
node ace migration:run
```

通过访问[官方文档](https://lucid.adonisjs.com/docs/migrations)了解更多关于迁移的信息。

## 查询构建器

Lucid 附带了一个基于 Knex 构建的流畅查询构建器。您可以使用查询构建器对数据库执行 CRUD 操作。

```ts
import db from '@adonisjs/lucid/services/db'

/**
 * 创建查询构建器实例
 */
const query = db.query()

/**
 * 创建查询构建器实例并选择表
 */
const queryWithTableSelection = db.from('users')
```

查询构建器也可以限定到模型实例。

```ts
import User from '#models/user'

const user = await User.query().where('username', 'rlanz').first()
```

通过访问[官方文档](https://lucid.adonisjs.com/docs/select-query-builder)了解更多关于查询构建器的信息。

## CRUD 操作

Lucid 模型具有内置方法来对数据库执行 CRUD 操作。

```ts
import User from '#models/user'

/**
 * 创建新用户
 */
const user = await User.create({
  username: 'rlanz',
  email: 'romain@adonisjs.com',
})

/**
 * 通过主键查找用户
 */
const user = await User.find(1)

/**
 * 更新用户
 */

const user = await User.find(1)
user.username = 'romain'
await user.save()

/**
 * 删除用户
 */
const user = await User.find(1)
await user.delete()
```

通过访问[官方文档](https://lucid.adonisjs.com/docs/crud-operations)了解更多关于 CRUD 操作的信息。

## 了解更多

- [Lucid 文档](https://lucid.adonisjs.com)
- [安装和使用](https://lucid.adonisjs.com/docs/installation)
- [CRUD 操作](https://lucid.adonisjs.com/docs/crud-operations)
- [模型钩子](https://lucid.adonisjs.com/docs/model-hooks)
- [关系](https://lucid.adonisjs.com/docs/relationships)
- [Adocasts Lucid 系列](https://adocasts.com/topics/lucid)
