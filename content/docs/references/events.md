---
summary: 了解 AdonisJS 框架核心和官方包派发的事件。
---

# 事件参考

在本指南中，我们将查看框架核心和官方包派发的事件列表。查看 [事件发射器](../digging_deeper/emitter.md) 文档以了解其用法。

## http\:request_completed

在 HTTP 请求完成后会派发 [`http:request_completed`](https://github.com/adonisjs/http-server/blob/main/src/types/server.ts#L65) 事件。该事件包含 [HttpContext](../concepts/http_context.md) 实例和请求持续时间。`duration` 值是 `process.hrtime` 方法的输出。

```ts
import emitter from '@adonisjs/core/services/emitter'
import string from '@adonisjs/core/helpers/string'

emitter.on('http:request_completed', (event) => {
  const method = event.ctx.request.method()
  const url = event.ctx.request.url(true)
  const duration = event.duration

  console.log(`${method} ${url}: ${string.prettyHrTime(duration)}`)
})
```

## http\:server_ready

当 AdonisJS HTTP 服务器准备好接受传入请求时，会派发该事件。

```ts
import emitter from '@adonisjs/core/services/emitter'

emitter.on('http:server_ready', (event) => {
  console.log(event.host)
  console.log(event.port)

  /**
   * 启动应用程序和 HTTP 服务器
   * 所花费的时间。
   */
  console.log(event.duration)
})
```

## container_binding\:resolved

在 IoC 容器解析绑定或构造类实例后，会派发该事件。`event.binding` 属性将是字符串（绑定名称）或类构造函数，`event.value` 属性是解析的值。

```ts
import emitter from '@adonisjs/core/services/emitter'

emitter.on('container_binding:resolved', (event) => {
  console.log(event.binding)
  console.log(event.value)
})
```

## session\:initiated

`@adonisjs/session` 包在 HTTP 请求期间启动会话存储时会发出该事件。`event.session` 属性是 [Session 类](https://github.com/adonisjs/session/blob/main/src/session.ts) 的实例。

```ts
import emitter from '@adonisjs/core/services/emitter'

emitter.on('session:initiated', (event) => {
  console.log(`为 ${event.session.sessionId} 启动存储`)
})
```

## session\:committed

`@adonisjs/session` 包在 HTTP 请求期间将会话数据写入会话存储时会发出该事件。

```ts
import emitter from '@adonisjs/core/services/emitter'

emitter.on('session:committed', (event) => {
  console.log(`为 ${event.session.sessionId} 持久化数据`)
})
```

## session\:migrated

使用 `session.regenerate()` 方法生成新会话 ID 时，`@adonisjs/session` 包会发出该事件。

```ts
import emitter from '@adonisjs/core/services/emitter'

emitter.on('session:migrated', (event) => {
  console.log(`将数据迁移到 ${event.toSessionId}`)
  console.log(`销毁会话 ${event.fromSessionId}`)
})
```

## i18n\:missing\:translation

当特定键和语言环境的翻译缺失时，`@adonisjs/i18n` 包会派发该事件。您可以监听此事件以查找给定语言环境的缺失翻译。

```ts
import emitter from '@adonisjs/core/services/emitter'

emitter.on('i18n:missing:translation', function (event) {
  console.log(event.identifier)
  console.log(event.hasFallback)
  console.log(event.locale)
})
```

## mail\:sending

在发送电子邮件之前，`@adonisjs/mail` 包会发出该事件。对于 `mail.sendLater` 方法调用，处理邮件队列作业时会发出该事件。

```ts
import emitter from '@adonisjs/core/services/emitter'

emitter.on('mail:sending', (event) => {
  console.log(event.mailerName)
  console.log(event.message)
  console.log(event.views)
})
```

## mail\:sent

发送电子邮件后，该事件由 `@adonisjs/mail` 包派发。

```ts
import emitter from '@adonisjs/core/services/emitter'

emitter.on('mail:sent', (event) => {
  console.log(event.response)

  console.log(event.mailerName)
  console.log(event.message)
  console.log(event.views)
})
```

## mail\:queueing

在将作业排队发送电子邮件之前，`@adonisjs/mail` 包会发出该事件。

```ts
import emitter from '@adonisjs/core/services/emitter'

emitter.on('mail:queueing', (event) => {
  console.log(event.mailerName)
  console.log(event.message)
  console.log(event.views)
})
```

## mail\:queued

邮件已排队后，该事件由 `@adonisjs/mail` 包派发。

```ts
import emitter from '@adonisjs/core/services/emitter'

emitter.on('mail:queued', (event) => {
  console.log(event.mailerName)
  console.log(event.message)
  console.log(event.views)
})
```

## queued\:mail\:error

当 `@adonisjs/mail` 包的 [MemoryQueue](https://github.com/adonisjs/mail/blob/main/src/messengers/memory_queue.ts) 实现无法发送使用 `mail.sendLater` 方法排队的电子邮件时，会派发该事件。

如果您使用的是自定义队列实现，则必须捕获作业错误并发出此事件。

```ts
import emitter from '@adonisjs/core/services/emitter'

emitter.on('queued:mail:error', (event) => {
  console.log(event.error)
  console.log(event.mailerName)
})
```

## session_auth\:login_attempted

当调用 `auth.login` 方法（直接或由会话守卫内部调用）时，`@adonisjs/auth` 包的 [SessionGuard](https://github.com/adonisjs/auth/blob/main/src/guards/session/guard.ts) 实现会派发该事件。

```ts
import emitter from '@adonisjs/core/services/emitter'

emitter.on('session_auth:login_attempted', (event) => {
  console.log(event.guardName)
  console.log(event.user)
})
```

## session_auth\:login_succeeded

用户成功登录后，`@adonisjs/auth` 包的 [SessionGuard](https://github.com/adonisjs/auth/blob/main/src/guards/session/guard.ts) 实现会派发该事件。

您可以使用此事件跟踪与给定用户关联的会话。

```ts
import emitter from '@adonisjs/core/services/emitter'

emitter.on('session_auth:login_succeeded', (event) => {
  console.log(event.guardName)
  console.log(event.sessionId)
  console.log(event.user)
  console.log(event.rememberMeToken) // （如果创建了一个）
})
```

## session_auth\:authentication_attempted

当尝试验证请求会话并检查已登录用户时，`@adonisjs/auth` 包会派发该事件。

```ts
import emitter from '@adonisjs/core/services/emitter'

emitter.on('session_auth:authentication_attempted', (event) => {
  console.log(event.guardName)
  console.log(event.sessionId)
})
```

## session_auth\:authentication_succeeded

验证请求会话并且用户已登录后，`@adonisjs/auth` 包会派发该事件。您可以使用 `event.user` 属性访问已登录用户。

```ts
import emitter from '@adonisjs/core/services/emitter'

emitter.on('session_auth:authentication_succeeded', (event) => {
  console.log(event.guardName)
  console.log(event.sessionId)

  console.log(event.user)
  console.log(event.rememberMeToken) // 如果使用令牌认证
})
```

## session_auth\:authentication_failed

当认证检查失败并且用户在当前 HTTP 请求期间未登录时，`@adonisjs/auth` 包会派发该事件。

```ts
import emitter from '@adonisjs/core/services/emitter'

emitter.on('session_auth:authentication_failed', (event) => {
  console.log(event.guardName)
  console.log(event.sessionId)

  console.log(event.error)
})
```

## session_auth\:logged_out

用户注销后，`@adonisjs/auth` 包会派发该事件。

```ts
import emitter from '@adonisjs/core/services/emitter'

emitter.on('session_auth:logged_out', (event) => {
  console.log(event.guardName)
  console.log(event.sessionId)

  /**
   * 当在没有用户登录的请求期间
   * 调用注销时，用户的值将为 null。
   */
  console.log(event.user)
})
```

## access_tokens_auth\:authentication_attempted

当尝试验证 HTTP 请求期间的访问令牌时，`@adonisjs/auth` 包会派发该事件。

```ts
import emitter from '@adonisjs/core/services/emitter'

emitter.on('access_tokens_auth:authentication_attempted', (event) => {
  console.log(event.guardName)
})
```

## access_tokens_auth\:authentication_succeeded

验证访问令牌后，`@adonisjs/auth` 包会派发该事件。您可以使用 `event.user` 属性访问已认证用户。

```ts
import emitter from '@adonisjs/core/services/emitter'

emitter.on('access_tokens_auth:authentication_succeeded', (event) => {
  console.log(event.guardName)
  console.log(event.user)
  console.log(event.token)
})
```

## access_tokens_auth\:authentication_failed

当认证检查失败时，`@adonisjs/auth` 包会派发该事件。

```ts
import emitter from '@adonisjs/core/services/emitter'

emitter.on('access_tokens_auth:authentication_failed', (event) => {
  console.log(event.guardName)
  console.log(event.error)
})
```

## authorization\:finished

执行授权检查后，`@adonisjs/bouncer` 包会派发该事件。事件负载包括您可以检查以了解检查状态的最终响应。

```ts
import emitter from '@adonisjs/core/services/emitter'

emitter.on('authorization:finished', (event) => {
  console.log(event.user)
  console.log(event.response)
  console.log(event.parameters)
  console.log(event.action)
})
```

## cache\:cleared

使用 `cache.clear` 方法清除缓存后，`@adonisjs/cache` 包会派发该事件。

```ts
import emitter from '@adonisjs/core/services/emitter'

emitter.on('cache:cleared', (event) => {
  console.log(event.store)
})
```

## cache\:deleted

使用 `cache.delete` 方法删除缓存键后，`@adonisjs/cache` 包会派发该事件。

```ts
import emitter from '@adonisjs/core/services/emitter'

emitter.on('cache:deleted', (event) => {
  console.log(event.key)
})
```

## cache\:hit

在缓存存储中找到缓存键时，`@adonisjs/cache` 包会派发该事件。

```ts
import emitter from '@adonisjs/core/services/emitter'

emitter.on('cache:hit', (event) => {
  console.log(event.key)
  console.log(event.value)
})
```

## cache\:miss

当缓存存储中找不到缓存键时，`@adonisjs/cache` 包会派发该事件。

```ts
import emitter from '@adonisjs/core/services/emitter'

emitter.on('cache:miss', (event) => {
  console.log(event.key)
})
```

## cache\:written

将缓存键写入缓存存储后，`@adonisjs/cache` 包会派发该事件。

```ts
import emitter from '@adonisjs/core/services/emitter'

emitter.on('cache:written', (event) => {
  console.log(event.key)
  console.log(event.value)
})
```
