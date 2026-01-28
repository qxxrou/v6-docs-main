---
summary: 使用 @adonisjs/session 包在 AdonisJS 应用程序中管理用户会话。
---

# 会话

您可以使用 `@adonisjs/session` 包在 AdonisJS 应用程序中管理用户会话。会话包提供了一个统一的 API，用于在不同的存储提供程序之间存储会话数据。

**以下是捆绑的存储列表：**

- `cookie`：会话数据存储在加密的 cookie 中。Cookie 存储在多服务器部署中表现良好，因为数据与客户端一起存储。

- `file`：会话数据保存在服务器上的文件中。只有在负载均衡器实现粘性会话的情况下，文件存储才能扩展到多服务器部署。

- `redis`：会话数据存储在 Redis 数据库中。Redis 存储推荐给具有大量会话数据的应用程序，并且可以扩展到多服务器部署。

- `dynamodb`：会话数据存储在 Amazon DynamoDB 表中。DynamoDB 存储适用于需要高度可扩展和分布式会话存储的应用程序，特别是当基础设施建立在 AWS 上时。

- `database`：会话数据使用 Lucid 存储在 SQL 数据库中。如果您已经在应用程序中使用 SQL 数据库，并且希望避免添加 Redis 等额外依赖项，数据库存储是一个不错的选择。

- `memory`：会话数据存储在全局内存存储中。内存存储在测试期间使用。

除了内置的后端存储之外，您还可以创建和[注册自定义会话存储](#creating-a-custom-session-store)。

## 安装

使用以下命令安装和配置包：

```sh
node ace add @adonisjs/session
```

:::disclosure{title="查看 add 命令执行的步骤"}

1. 使用检测到的包管理器安装 `@adonisjs/session` 包。

2. 在 `adonisrc.ts` 文件中注册以下服务提供程序和命令。

   ```ts
   {
     commands: [
       // ...其他命令
       () => import('@adonisjs/session/commands')
     ],
     providers: [
       // ...其他提供程序
       () => import('@adonisjs/session/session_provider')
     ]
   }
   ```

3. 创建 `config/session.ts` 文件。

4. 定义以下环境变量及其验证。

   ```dotenv
   SESSION_DRIVER=cookie
   ```

5. 在 `start/kernel.ts` 文件中注册以下中间件。

   ```ts
   router.use([() => import('@adonisjs/session/session_middleware')])
   ```

:::

## 配置

会话包的配置存储在 `config/session.ts` 文件中。

另请参阅：[会话配置存根](https://github.com/adonisjs/session/blob/main/stubs/config/session.stub)

```ts
import env from '#start/env'
import app from '@adonisjs/core/services/app'
import { defineConfig, stores } from '@adonisjs/session'

export default defineConfig({
  age: '2h',
  enabled: true,
  cookieName: 'adonis-session',
  clearWithBrowser: false,

  cookie: {
    path: '/',
    httpOnly: true,
    secure: app.inProduction,
    sameSite: 'lax',
  },

  store: env.get('SESSION_DRIVER'),
  stores: {
    cookie: stores.cookie(),
  },
})
```

<dl>

<dt>

enabled

</dt>

<dd>

启用或禁用中间件，而无需将其从中间件堆栈中删除。

</dd>

<dt>

cookieName

</dt>

<dd>

Cookie 名称用于存储会话 ID。可以随意重命名。

</dd>

<dt>

clearWithBrowser

</dt>

<dd>

当设置为 true 时，用户关闭浏览器窗口后，会话 ID cookie 将被删除。此 cookie 技术上称为[会话 cookie](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies#define_the_lifetime_of_a_cookie)。

</dd>

<dt>

age

</dt>

<dd>

`age` 属性控制没有用户活动时会话数据的有效期。在给定持续时间后，会话数据被视为过期。

</dd>

<dt>

cookie

</dt>

<dd>

控制会话 ID cookie 属性。另请参阅 [cookie 配置](./cookies.md#configuration)。

</dd>

<dt>

store

</dt>

<dd>

定义要用于存储会话数据的存储。它可以是一个固定值，也可以从环境变量中读取。

</dd>

<dt>

stores

</dt>

<dd>

`stores` 对象用于配置一个或多个后端存储。

大多数应用程序将使用单个存储。但是，您可以配置多个存储，并根据应用程序运行的环境在它们之间切换。

</dd>

</dl>

---

### 存储配置

以下是 `@adonisjs/session` 包捆绑的后端存储列表。

```ts
import app from '@adonisjs/core/services/app'
import { defineConfig, stores } from '@adonisjs/session'

export default defineConfig({
  store: env.get('SESSION_DRIVER'),

  // highlight-start
  stores: {
    cookie: stores.cookie(),

    file: stores.file({
      location: app.tmpPath('sessions')
    }),

    redis: stores.redis({
      connection: 'main'
    })

    dynamodb: stores.dynamodb({
      clientConfig: {}
    }),

    database: stores.database({
      connectionName: 'postgres',
      tableName: 'sessions',
    }),
  }
  // highlight-end
})
```

<dl>

<dt>

stores.cookie

</dt>

<dd>

`cookie` 存储加密并将会话数据存储在 cookie 中。

</dd>

<dt>

stores.file

</dt>

<dd>

定义 `file` 存储的配置。该方法接受用于存储会话文件的 `location` 路径。

</dd>

<dt>

stores.redis

</dt>

<dd>

定义 `redis` 存储的配置。该方法接受用于存储会话数据的 `connection` 名称。

在使用 `redis` 存储之前，请确保首先安装和配置 [@adonisjs/redis](../database/redis.md) 包。

</dd>

<dt>

stores.dynamodb

</dt>

<dd>

定义 `dynamodb` 存储的配置。您可以通过 `clientConfig` 属性传递 [DynamoDB 配置](https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/Package/-aws-sdk-client-dynamodb/Interface/DynamoDBClientConfig/)，或者将 DynamoDB 的实例作为 `client` 属性传递。

```ts
// title: 使用客户端配置
stores.dynamodb({
  clientConfig: {
    region: 'us-east-1',
    endpoint: '<database-endpoint>',
    credentials: {
      accessKeyId: '',
      secretAccessKey: '',
    },
  },
})
```

```ts
// title: 使用客户端实例
import { DynamoDBClient } from '@aws-sdk/client-dynamodb'
const client = new DynamoDBClient({})

stores.dynamodb({
  client,
})
```

此外，您可以定义自定义表名和键属性名。

```ts
stores.dynamodb({
  tableName: 'Session'
  keyAttribute: 'key'
})
```

</dd>

<dt>

stores.database

</dt>

<dd>

定义 `database` 存储的配置。该方法可选地接受要使用的 `connectionName` 和用于存储会话数据的 `tableName`。

在使用 `database` 存储之前，请确保首先安装和配置 [@adonisjs/lucid](../database/lucid.md) 包。

您还必须在数据库内创建会话表。您可以使用以下命令创建迁移文件。

```sh
node ace make:session-table
```

```ts
stores.database({
  connectionName: 'postgres',
  tableName: 'sessions',
  gcProbability: 2,
})
```

与 Redis 不同，Redis 会自动处理过期，SQL 数据库需要手动清理过期的会话。数据库存储实现了概率垃圾收集：在每个请求上，有 `gcProbability` 百分比的机会（默认：**2%**）从数据库中删除过期的会话。

将 `gcProbability` 设置为 `0` 以完全禁用自动垃圾收集。在这种情况下，您应该设置一个计划任务来定期清理过期的会话。

#### 向现有数据库添加标记支持

如果您有一个现有的会话表并希望使用[会话标记](#session-tagging)，您需要添加 `user_id` 列。创建一个新的迁移：

```sh
node ace make:migration add_user_id_to_sessions
```

```ts
import { BaseSchema } from '@adonisjs/lucid/schema'

export default class extends BaseSchema {
  protected tableName = 'sessions'

  async up() {
    this.schema.alterTable(this.tableName, (table) => {
      table.string('user_id').nullable().index()
    })
  }

  async down() {
    this.schema.alterTable(this.tableName, (table) => {
      table.dropColumn('user_id')
    })
  }
}
```

</dd>

</dl>

---

### 更新环境变量验证

如果您决定使用除默认存储之外的会话存储，请确保也更新 `SESSION_DRIVER` 环境变量的环境变量验证。

在以下示例中，我们配置了 `cookie`、`redis` 和 `dynamodb` 存储。因此，我们也应该允许 `SESSION_DRIVER` 环境变量是其中之一。

```ts
import { defineConfig, stores } from '@adonisjs/session'

export default defineConfig({
  // highlight-start
  store: env.get('SESSION_DRIVER'),

  stores: {
    cookie: stores.cookie(),
    redis: stores.redis({
      connection: 'main',
    }),
  },
  // highlight-end
})
```

```ts
// title: start/env.ts
{
  SESSION_DRIVER: Env.schema.enum(['cookie', 'redis', 'memory'] as const)
}
```

## 基本示例

一旦会话包已注册，您就可以从 [HTTP 上下文](../concepts/http_context.md)访问 `session` 属性。会话属性公开了用于读取和写入会话存储数据的 API。

```ts
import router from '@adonisjs/core/services/router'

router.get('/theme/:color', async ({ params, session, response }) => {
  // highlight-start
  session.put('theme', params.color)
  // highlight-end
  response.redirect('/')
})

router.get('/', async ({ session }) => {
  // highlight-start
  const colorTheme = session.get('theme')
  // highlight-end
  return `您正在使用 ${colorTheme} 颜色主题`
})
```

会话数据在请求开始时从会话存储中读取，并在结束时写回存储。因此，所有更改都保存在内存中，直到请求完成。

## 支持的数据类型

会话数据使用 `JSON.stringify` 序列化为字符串；因此，您可以将以下 JavaScript 数据类型用作会话值。

- string
- number
- bigInt
- boolean
- null
- object
- array

```ts
// 对象
session.put('user', {
  id: 1,
  fullName: 'virk',
})

// 数组
session.put('product_ids', [1, 2, 3, 4])

// 布尔值
session.put('is_logged_in', true)

// 数字
session.put('visits', 10)

// BigInt
session.put('visits', BigInt(10))

// 数据对象转换为 ISO 字符串
session.put('visited_at', new Date())
```

## 读取和写入数据

以下是可以用于与 `session` 对象中的数据交互的方法列表。

### get

从存储中返回键的值。您可以使用点表示法读取嵌套值。

```ts
session.get('key')
session.get('user.email')
```

您还可以将默认值定义为第二个参数。如果键在存储中不存在，将返回默认值。

```ts
session.get('visits', 0)
```

### has

检查会话存储中是否存在键。

```ts
if (session.has('visits')) {
}
```

### all

返回会话存储中的所有数据。返回值将始终是一个对象。

```ts
session.all()
```

### put

向会话存储添加键值对。您可以使用点表示法创建具有嵌套值的对象。

```ts
session.put('user', { email: 'foo@bar.com' })

// 与上面相同
session.put('user.email', 'foo@bar.com')
```

### forget

从会话存储中删除键值对。

```ts
session.forget('user')

// 从用户对象中删除电子邮件
session.forget('user.email')
```

### pull

`pull` 方法返回键的值，并同时将其从存储中删除。

```ts
const user = session.pull('user')
session.has('user') // false
```

### increment

`increment` 方法递增键的值。如果键尚不存在，则定义新的键值。

```ts
session.increment('visits')

// 递增 4
session.increment('visits', 4)
```

### decrement

`decrement` 方法递减键的值。如果键尚不存在，则定义新的键值。

```ts
session.decrement('visits')

// 递减 4
session.decrement('visits', 4)
```

### clear

`clear` 方法从会话存储中删除所有内容。

```ts
session.clear()
```

## 会话生命周期

AdonisJS 创建一个空的会话存储，并在第一个 HTTP 请求上为其分配一个唯一的会话 ID，即使请求/响应生命周期不与会话交互也是如此。

在随后的每个请求上，我们更新会话 ID cookie 的 `maxAge` 属性，以确保它不会过期。会话存储也会收到有关更改的通知（如果有），以更新和持久化它们。

您可以使用 `sessionId` 属性访问唯一的会话 ID。访问者的会话 ID 保持不变，直到过期。

```ts
console.log(session.sessionId)
```

### 重新生成会话 ID

重新生成会话 ID 有助于防止应用程序中的[会话固定](https://owasp.org/www-community/attacks/Session_fixation)攻击。在将匿名会话与登录用户关联时，必须重新生成会话 ID。

`@adonisjs/auth` 包会自动重新生成会话 ID，因此您无需手动执行。

```ts
/**
 * 新的会话 ID 将在
 * 请求结束时分配
 */
session.regenerate()
```

## 会话标记

会话标记允许您将会话与用户 ID 关联。这对于实现以下功能很有用：

- **从所有设备注销**：销毁与用户关联的所有会话。
- **活动会话列表**：在用户的账户设置中显示所有活动会话。

:::note
会话标记仅受 `redis`、`database` 和 `memory` 存储支持。尝试将标记与 `cookie`、`file` 或 `dynamodb` 存储一起使用将抛出错误。
:::

### 标记当前会话

您可以使用 `session.tag` 方法使用用户 ID 标记当前会话。这通常在用户登录后完成。

```ts
import router from '@adonisjs/core/services/router'

router.post('/login', async ({ auth, session }) => {
  const user = await auth.use('web').login(email, password)

  // highlight-start
  // 使用用户 ID 标记会话
  await session.tag(String(user.id))
  // highlight-end
})
```

## SessionCollection

`SessionCollection` 类提供了在 HTTP 请求上下文之外以编程方式管理会话的 API。它允许您通过其 ID 读取、销毁和标记会话。

### 创建实例

您必须通过容器获取 `SessionCollection` 实例，以确保正确的配置注入。

```ts
import app from '@adonisjs/core/services/app'
import { SessionCollection } from '@adonisjs/session'

const sessionCollection = await app.container.make(SessionCollection)
```

### 可用方法

```ts
// 按 ID 获取会话数据
const data = await sessionCollection.get(sessionId)

// 按 ID 销毁会话
await sessionCollection.destroy(sessionId)

// 使用用户 ID 标记会话
await sessionCollection.tag(sessionId, userId)

// 获取用户的所有会话
const sessions = await sessionCollection.tagged(userId)
// 返回：Array<{ id: string, data: SessionData }>

// 检查当前存储是否支持标记
const supportsTagging = sessionCollection.supportsTagging()
```

### 列出活动会话

以下是在用户账户设置页面中列出所有活动会话的示例。

```ts
import app from '@adonisjs/core/services/app'
import router from '@adonisjs/core/services/router'
import { SessionCollection } from '@adonisjs/session'

router.get('/account/sessions', async ({ auth, session, view }) => {
  const user = auth.user!
  const sessionCollection = await app.container.make(SessionCollection)

  // highlight-start
  const sessions = await sessionCollection.tagged(String(user.id))

  // 标记当前会话
  const sessionsWithCurrent = sessions.map((tagged) => ({
    ...tagged,
    isCurrent: tagged.id === session.sessionId,
  }))
  // highlight-end

  return view.render('account/sessions', { sessions: sessionsWithCurrent })
})
```

### 从所有其他设备注销

以下是实现"从所有其他设备注销"功能的完整示例。

```ts
import app from '@adonisjs/core/services/app'
import router from '@adonisjs/core/services/router'
import { SessionCollection } from '@adonisjs/session'

router.post('/logout-other-devices', async ({ auth, session, response }) => {
  const user = auth.user!
  const sessionCollection = await app.container.make(SessionCollection)

  // highlight-start
  const sessions = await sessionCollection.tagged(String(user.id))

  for (const s of sessions) {
    if (s.id !== session.sessionId) {
      await sessionCollection.destroy(s.id)
    }
  }
  // highlight-end

  return response.redirect().back()
})
```

## 闪现消息

闪现消息用于在两个 HTTP 请求之间传递数据。它们通常用于在特定操作后向用户提供反馈。例如，在表单提交后显示成功消息或显示验证错误消息。

在以下示例中，我们定义了用于显示联系表单和将表单详细信息提交到数据库的路由。表单提交后，我们使用闪现消息将用户重定向回表单以及成功通知。

```ts
import router from '@adonisjs/core/services/router'

router.post('/contact', ({ session, request, response }) => {
  const data = request.all()
  // 保存联系数据

  // highlight-start
  session.flash('notification', {
    type: 'success',
    message: '感谢您的联系。我们将与您联系',
  })
  // highlight-end

  response.redirect().back()
})

router.get('/contact', ({ view }) => {
  return view.render('contact')
})
```

您可以使用 `flashMessage` 标签或 `flashMessages` 属性在边缘模板中访问闪现消息。

```edge
@flashMessage('notification')
  <div class="notification {{ $message.type }}">
    {{ $message.message }}
  </div>
@end

<form method="POST" action="/contact">
  <!-- 表单的其余部分 -->
</form>
```

您可以使用 `session.flashMessages` 属性在控制器中访问闪现消息。

```ts
router.get('/contact', ({ view, session }) => {
  // highlight-start
  console.log(session.flashMessages.all())
  // highlight-end
  return view.render('contact')
})
```

### 验证错误和闪现消息

会话中间件自动捕获[验证异常](./validation.md#error-handling)并将用户重定向回表单。验证错误和表单输入数据保存在闪现消息中，您可以在 Edge 模板中访问它们。

在以下示例中：

- 我们使用 [`old` 方法](../references/edge.md#old) 访问 `title` 输入字段的值。
- 并使用 [`@inputError` 标签](../references/edge.md#inputerror) 访问错误消息。

```edge
<form method="POST" action="/posts">
  <div>
    <label for="title"> 标题 </label>
    <input
      type="text"
      id="title"
      name="title"
      value="{{ old('title') || '' }}"
    />

    @inputError('title')
      @each(message in $messages)
        <p> {{ message }} </p>
      @end
    @end
  </div>
</form>
```

### 写入闪现消息

以下是将数据写入闪现消息存储的方法。`session.flash` 方法接受键值对，并将其写入会话存储中的闪现消息属性。

```ts
session.flash('key', value)
session.flash({
  key: value,
})
```

您无需手动读取请求数据并将其存储在闪现消息中，而是可以使用以下方法之一来闪现表单数据。

```ts
// title: flashAll
/**
 * 闪现请求的简写
 * 数据
 */
session.flashAll()

/**
 * 与 "flashAll" 相同
 */
session.flash(request.all())
```

```ts
// title: flashOnly
/**
 * 闪现选定内容的简写
 * 来自请求数据的属性
 */
session.flashOnly(['username', 'email'])

/**
 * 与 "flashOnly" 相同
 */
session.flash(request.only(['username', 'email']))
```

```ts
// title: flashExcept
/**
 * 闪现选定内容的简写
 * 来自请求数据的属性
 */
session.flashExcept(['password'])

/**
 * 与 "flashExcept" 相同
 */
session.flash(request.except(['password']))
```

最后，您可以使用 `session.reflash` 方法重新闪现当前的闪现消息。

```ts
session.reflash()
session.reflashOnly(['notification', 'errors'])
session.reflashExcept(['errors'])
```

### 读取闪现消息

闪现消息仅在重定向后的后续请求中可用。您可以使用 `session.flashMessages` 属性访问它们。

```ts
console.log(session.flashMessages.all())
console.log(session.flashMessages.get('key'))
console.log(session.flashMessages.has('key'))
```

相同的 `flashMessages` 属性也与 Edge 模板共享，您可以通过以下方式访问它：

另请参阅：[Edge 助手参考](../references/edge.md#flashmessages)

```edge
{{ flashMessages.all() }}
{{ flashMessages.get('key') }}
{{ flashMessages.has('key') }}
```

最后，您可以使用以下 Edge 标签访问特定的闪现消息或验证错误。

```edge
{{-- 按键读取任何闪现消息 --}}
@flashMessage('key')
  {{ inspect($message) }}
@end

{{-- 读取通用错误 --}}
@error('key')
  {{ inspect($message) }}
@end

{{-- 读取验证错误 --}}
@inputError('key')
  {{ inspect($messages) }}
@end
```

## 事件

请查看[事件参考指南](../references/events.md#sessioninitiated)以查看 `@adonisjs/session` 包调度的事件列表。

## 创建自定义会话存储

会话存储必须实现 [SessionStoreContract](https://github.com/adonisjs/session/blob/main/src/types.ts#L23C18-L23C38) 接口并定义以下方法。

```ts
import {
  SessionData,
  SessionStoreFactory,
  SessionStoreContract,
} from '@adonisjs/session/types'

/**
 * 您要接受的配置
 */
export type MongoDBConfig = {}

/**
 * 驱动程序实现
 */
export class MongoDBStore implements SessionStoreContract {
  constructor(public config: MongoDBConfig) {}

  /**
   * 返回会话 ID 的会话数据。该方法
   * 必须返回 null 或键值对的对象
   */
  async read(sessionId: string): Promise<SessionData | null> {}

  /**
   * 保存会话数据以针对提供的会话 ID
   */
  async write(sessionId: string, data: SessionData): Promise<void> {}

  /**
   * 删除给定会话 ID 的会话数据
   */
  async destroy(sessionId: string): Promise<void> {}

  /**
   * 重置会话过期时间
   */
  async touch(sessionId: string): Promise<void> {}
}

/**
 * 工厂函数以引用存储
 * 在配置文件内。
 */
export function mongoDbStore(config: MongoDbConfig): SessionStoreFactory {
  return (ctx, sessionConfig) => {
    return new MongoDBStore(config)
  }
}
```

在上面的代码示例中，我们导出以下值。

- `MongoDBConfig`：您要接受的配置的 TypeScript 类型。

- `MongoDBStore`：存储的实现作为类。它必须遵守 `SessionStoreContract` 接口。

- `mongoDbStore`：最后，一个工厂函数，用于为每个 HTTP 请求创建存储的实例。

### 使用存储

创建存储后，您可以使用 `mongoDbStore` 工厂函数在配置文件内引用它。

```ts
// title: config/session.ts
import { defineConfig } from '@adonisjs/session'
import { mongDbStore } from 'my-custom-package'

export default defineConfig({
  stores: {
    mongodb: mongoDbStore({
      // 配置在这里
    }),
  },
})
```

### 关于序列化数据的说明

`write` 方法将会话数据作为对象接收，您可能必须在保存之前将其转换为字符串。您可以使用任何序列化包，或者使用 AdonisJS 助手模块提供的 [MessageBuilder](../references/helpers.md#message-builder) 助手。有关灵感，请咨询官方的[会话存储](https://github.com/adonisjs/session/blob/main/src/stores/redis.ts#L59)。
