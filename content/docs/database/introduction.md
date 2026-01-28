---
summary: AdonisJS 应用中 SQL 库和 ORM 的可用选项。
---

# SQL 和 ORM

SQL 数据库在持久化存储中存储应用数据方面非常受欢迎。您可以在 AdonisJS 应用中使用任何库和 ORM 来执行 SQL 查询。

:::note
AdonisJS 核心团队构建了 [Lucid ORM](./lucid.md)，但并不强制您使用它。您可以在 AdonisJS 应用中使用任何其他您喜欢的 SQL 库和 ORM。
:::

## 热门选项

以下是您可以在 AdonisJS 应用中使用的其他热门 SQL 库和 ORM 列表（就像在任何其他 Node.js 应用中一样）。

- [**Lucid**](./lucid.md) 是一个 SQL 查询构建器和 **Active Record ORM**，构建在 [Knex](https://knexjs.org) 之上，由 AdonisJS 核心团队创建和维护。
- [**Prisma**](https://prisma.io/orm) Prisma ORM 是 Node.js 生态系统中另一个流行的 ORM。它拥有庞大的社区支持。它提供直观的数据模型、自动迁移、类型安全和自动补全功能。
- [**Kysely**](https://kysely.dev/docs/getting-started) 是 Node.js 的端到端类型安全查询构建器。如果您需要一个没有模型的精简查询构建器，Kysely 是一个很好的选择。我们写了一篇文章解释[如何在 AdonisJS 应用中集成 Kysely](https://adonisjs.com/blog/kysely-with-adonisjs)。
- [**Drizzle ORM**](https://orm.drizzle.team/) 被我们的社区中的许多 AdonisJS 开发者使用。我们没有使用这个 ORM 的经验，但您可能想要尝试一下，看看它是否适合您的用例。
- [**Mikro ORM**](https://mikro-orm.io/docs/guide/first-entity) 是 Node.js 生态系统中一个被低估的 ORM。与 Lucid 相比，MikroORM 有点冗长。然而，它正在积极维护，并且也构建在 Knex 之上。
- [**TypeORM**](https://typeorm.io) 是 TypeScript 生态系统中一个流行的 ORM。

## 使用其他 SQL 库和 ORM

当使用其他 SQL 库或 ORM 时，您将必须手动更改某些包的配置。

### 身份验证

[AdonisJS 身份验证模块](../authentication/introduction.md) 内置支持 Lucid 来获取经过身份验证的用户。当使用其他 SQL 库或 ORM 时，您将必须实现 `SessionUserProviderContract` 或 `AccessTokensProviderContract` 接口来获取用户。

以下是使用 `Kysely` 时如何实现 `SessionUserProviderContract` 接口的示例。

```ts
import { symbols } from '@adonisjs/auth'
import type {
  SessionGuardUser,
  SessionUserProviderContract,
} from '@adonisjs/auth/types/session'
import type { Users } from '../../types/db.js' // Kysely 特有

export class SessionKyselyUserProvider implements SessionUserProviderContract<Users> {
  /**
   * 事件发射器用于为会话守卫发出的事件添加类型信息。
   */
  declare [symbols.PROVIDER_REAL_USER]: Users

  /**
   * 会话守卫和您的提供者之间的桥梁。
   */
  async createUserForGuard(user: Users): Promise<SessionGuardUser<Users>> {
    return {
      getId() {
        return user.id
      },
      getOriginal() {
        return user
      },
    }
  }

  /**
   * 使用您的自定义 SQL 库或 ORM 通过用户 ID 查找用户。
   */
  async findById(identifier: number): Promise<SessionGuardUser<Users> | null> {
    const user = await db
      .selectFrom('users')
      .selectAll()
      .where('id', '=', identifier)
      .executeTakeFirst()

    if (!user) {
      return null
    }

    return this.createUserForGuard(user)
  }
}
```

实现 `UserProvider` 接口后，您可以在配置中使用它。

```ts
const authConfig = defineConfig({
  default: 'web',

  guards: {
    web: sessionGuard({
      useRememberMeTokens: false,
      provider: sessionUserProvider({
        model: () => import('#models/user'),
      }),

      provider: configProvider.create(async () => {
        const { SessionKyselyUserProvider } = await import(
          '../app/auth/session_user_provider.js' // 文件路径
        )

        return new SessionKyselyUserProvider()
      }),
    }),
  },
})
```
