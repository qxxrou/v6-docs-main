---
summary: 了解 AdonisJS 中的身份验证系统以及如何在你的应用程序中验证用户身份。
---

# 身份验证

AdonisJS 提供了一个强大且安全的身份验证系统，可用于登录和验证应用程序的用户。无论是服务器渲染的应用程序、SPA 客户端还是移动应用，你都可以为它们设置身份验证。

身份验证包围绕 **guards（守卫）** 和 **providers（提供者）** 构建。

- Guards 是特定登录类型的端到端实现。例如，`session` 守卫允许你使用 cookies 和 session 来验证用户身份。而 `access_tokens` 守卫则允许你使用令牌来验证客户端。

- Providers 用于从数据库中查找用户和令牌。你可以使用内置的 providers，也可以实现自己的 providers。

:::note

为了确保应用程序的安全性，我们会正确地对用户密码和令牌进行哈希处理。此外，AdonisJS 的安全原语还受到保护，可防止[时序攻击](https://en.wikipedia.org/wiki/Timing_attack)和[会话固定攻击](https://owasp.org/www-community/attacks/Session_fixation)。

:::

## Auth 包不支持的功能

auth 包仅专注于验证 HTTP 请求，以下功能不在其范围内：

- 用户注册功能，如**注册表单**、**电子邮件验证**和**账户激活**。
- 账户管理功能，如**密码恢复**或**电子邮件更新**。
- 分配角色或验证权限。相反，请[使用 bouncer](../security/authorization.md) 在应用程序中实现授权检查。

<!-- :::note

**正在寻找功能齐全的用户管理系统？**\

请查看 persona。Persona 是一个官方包和入门套件，提供功能齐全的用户管理系统。

它提供了用户注册、电子邮件管理、会话跟踪、个人资料管理和双因素身份验证的即用操作。

::: -->

## 选择身份验证守卫

以下内置身份验证守卫为你提供了最简单的用户身份验证工作流程，同时不会牺牲应用程序的安全性。此外，你还可以[构建自己的身份验证守卫](./custom_auth_guard.md)以满足自定义需求。

### Session（会话）

会话守卫使用 [@adonisjs/session](../basics/session.md) 包在会话存储中跟踪已登录用户的状态。

会话和 cookies 在互联网上已经存在很长时间，并且适用于大多数应用程序。我们建议在以下情况下使用会话守卫：

- 如果你正在创建服务器渲染的 Web 应用程序。
- 或者，AdonisJS API 与其客户端位于同一顶级域。例如，`api.example.com` 和 `example.com`。

### Access tokens（访问令牌）

访问令牌是加密安全的随机令牌（也称为不透明访问令牌），在用户成功登录后颁发给用户。你可以在 AdonisJS 服务器无法写入/读取 cookies 的应用程序中使用访问令牌。例如：

- 原生移动应用。
- 托管在与 AdonisJS API 服务器不同域上的 Web 应用程序。

使用访问令牌时，客户端应用程序有责任安全地存储它们。访问令牌提供对应用程序的无限制访问（代表用户），泄露它们可能会导致安全问题。

### Basic auth（基本身份验证）

基本身份验证守卫是 [HTTP 身份验证框架](https://developer.mozilla.org/en-US/docs/Web/HTTP/Authentication) 的实现，客户端必须通过 `Authorization` 标头传递用户凭据作为 base64 编码字符串。

基本身份验证并不是实现安全登录系统的最佳方式。但是，你可以在应用程序处于活跃开发阶段时临时使用它。

## 选择用户提供者

如前所述，用户提供者负责在身份验证过程中查找用户。

用户提供者是特定于守卫的；例如，会话守卫的用户提供者负责通过其 ID 查找用户，而访问令牌守卫的用户提供者也负责验证访问令牌。

我们为内置守卫提供了一个 Lucid 用户提供者，该提供者使用 Lucid 模型来查找用户、生成令牌和验证令牌。

<!-- 如果你不使用 Lucid，则必须[实现自定义用户提供者]()。 -->

## 安装

身份验证系统已预先配置在 `web` 和 `api` 入门套件中。但是，你也可以在应用程序中手动安装和配置它，如下所示。

```sh
# 使用会话守卫配置（默认）
node ace add @adonisjs/auth --guard=session

# 使用访问令牌守卫配置
node ace add @adonisjs/auth --guard=access_tokens

# 使用基本身份验证守卫配置
node ace add @adonisjs/auth --guard=basic_auth
```

:::disclosure{title="查看 add 命令执行的步骤"}

1. 使用检测到的包管理器安装 `@adonisjs/auth` 包。

2. 在 `adonisrc.ts` 文件中注册以下服务提供者。

   ```ts
   {
     providers: [
       // ...其他提供者
       () => import('@adonisjs/auth/auth_provider'),
     ]
   }
   ```

3. 在 `start/kernel.ts` 文件中创建并注册以下中间件。

   ```ts
   router.use([() => import('@adonisjs/auth/initialize_auth_middleware')])
   ```

   ```ts
   router.named({
     auth: () => import('#middleware/auth_middleware'),
     // 仅在使用会话守卫时
     guest: () => import('#middleware/guest_middleware'),
   })
   ```

4. 在 `app/models` 目录中创建用户模型。
5. 为 `users` 表创建数据库迁移。
6. 为选定的守卫创建数据库迁移。
   :::

## 初始化身份验证中间件

在设置过程中，我们会在你的应用程序中注册 `@adonisjs/auth/initialize_auth_middleware`。该中间件负责创建 [Authenticator](https://github.com/adonisjs/auth/blob/main/src/authenticator.ts) 类的实例，并通过 `ctx.auth` 属性与请求的其余部分共享它。

请注意，初始化身份验证中间件不会验证请求或保护路由。它仅用于初始化身份验证器并与请求的其余部分共享它。你必须使用 [auth](./session_guard.md#protecting-routes) 中间件来保护路由。

此外，相同的身份验证器实例会与 Edge 模板共享（如果你的应用程序使用 Edge），你可以使用 `auth` 属性访问它。例如：

```edge
@if(auth.isAuthenticated)
  <p> 你好 {{ auth.user.email }} </p>
@end
```

## 创建用户表

`configure` 命令会在 `database/migrations` 目录中为 `users` 表创建数据库迁移。你可以随意打开此文件并根据应用程序需求进行更改。

默认情况下，会创建以下列。

```ts
import { BaseSchema } from '@adonisjs/lucid/schema'

export default class extends BaseSchema {
  protected tableName = 'users'

  async up() {
    this.schema.createTable(this.tableName, (table) => {
      table.increments('id').notNullable()
      table.string('full_name').nullable()
      table.string('email', 254).notNullable().unique()
      table.string('password').notNullable()

      table.timestamp('created_at').notNullable()
      table.timestamp('updated_at').nullable()
    })
  }

  async down() {
    this.schema.dropTable(this.tableName)
  }
}
```

此外，如果你定义、重命名或删除 `users` 表中的列，也请更新 `User` 模型。

## 后续步骤

- 了解如何在不牺牲应用程序安全性的情况下[验证用户凭据](./verifying_user_credentials.md)。
- 使用[会话守卫](./session_guard.md)进行有状态身份验证。
- 使用[访问令牌守卫](./access_tokens_guard.md)进行基于令牌的身份验证。
