---
summary: 为你的 AdonisJS 应用添加分布式追踪和可观测性。
---

# OpenTelemetry

本指南介绍 AdonisJS 应用中的 OpenTelemetry 集成。你将学习如何安装和配置 `@adonisjs/otel` 包，了解 OpenTelemetry 概念（如 trace 和 span），使用自动检测功能追踪 HTTP 请求和数据库查询，使用助手和装饰器创建自定义 span，跨服务传播追踪上下文，以及为生产环境配置采样和导出器。

## 概述

OpenTelemetry 是一个用于收集应用遥测数据的开放标准：追踪、指标和日志。`@adonisjs/otel` 包提供了 AdonisJS 和 OpenTelemetry 之间的无缝集成，为你提供分布式追踪和自动检测功能，具有合理的默认值和零配置设置。

可观测性对于了解应用内部发生的情况至关重要，尤其是在生产环境中。当用户报告"结账页面很慢"时，追踪功能可以让你准确地看到时间花费在哪里：是数据库查询？外部 API 调用？还是某个慢服务？没有追踪功能，你只能猜测。

:::media
![alt text](./tracing.png)
:::

这个包为你处理 OpenTelemetry 设置的复杂性。运行一个命令，你的应用就会自动追踪 HTTP 请求、数据库查询、Redis 操作等。

## OpenTelemetry 概念

在深入实现之前，你应该了解一些 OpenTelemetry 的核心概念。如需全面介绍，请参阅 [OpenTelemetry 官方文档](https://opentelemetry.io/docs/concepts/observability-primer/)。

**trace** 表示请求在系统中的完整旅程。当用户访问你的 API 时，trace 会捕获发生的一切：HTTP 请求、数据库查询、缓存查找、对外部服务的调用以及响应。

**span** 是 trace 中的单个工作单元。每个数据库查询、HTTP 请求或函数调用都可以是一个 span。span 有开始时间、持续时间、名称和属性（键值元数据）。span 是分层嵌套的：HTTP 请求的父 span 包含该请求期间进行的每个数据库查询的子 span。

**attribute** 是附加到 span 的键值对，提供上下文信息。例如，HTTP span 可能有 `http.method: GET`、`http.route: /users/:id` 和 `http.status_code: 200` 等属性。

## 安装

使用以下命令安装并配置包：

```sh
node ace add @adonisjs/otel
```

此命令会在项目根目录创建 `otel.ts` 文件（包含 OpenTelemetry 初始化代码），在 `bin/server.ts` 顶部添加导入语句，注册提供程序和中间件，并设置环境变量。

就是这样。你的应用现在已经对 HTTP 请求、数据库查询等进行自动追踪。

:::warning

**导入顺序至关重要**

OpenTelemetry 必须在任何其他代码加载之前初始化。SDK 需要在导入 `http`、`pg` 和 `redis` 等库之前对其进行修补。这就是为什么 `otel.ts` 作为第一行导入到 `bin/server.ts` 中的原因。

如果你移动或删除 `import './otel.js'` 行，自动检测功能将无法工作。你仍然可以创建手动 span，但 HTTP 请求和数据库查询的自动追踪将不会被捕获。

:::

## 手动设置

如果你不想使用 `node ace add`，以下是它的配置内容。

首先，在 `otel.ts` 创建一个文件，包含 OpenTelemetry 初始化代码：

```ts
// title: otel.ts
import { init } from '@adonisjs/otel/init'

await init(import.meta.dirname)
```

然后更新 `bin/server.ts`，将其作为第一行导入：

```ts
// title: bin/server.ts
/**
 * OpenTelemetry initialization - MUST be the first import
 */
import './otel.js'

import { Ignitor } from '@adonisjs/core'
// ... rest of your server setup
```

在 `adonisrc.ts` 中添加提供程序，以钩子到 AdonisJS 的异常处理程序并在 span 中记录错误：

```ts
// title: adonisrc.ts
{
  providers: [
    // ... other providers
    () => import('@adonisjs/otel/otel_provider'),
  ]
}
```

最后，在 `start/kernel.ts` 中将中间件添加为第一个路由器中间件，以丰富 HTTP span 的路由信息：

```ts
// title: start/kernel.ts
router.use([
  () => import('@adonisjs/otel/otel_middleware'),
  // ... other middlewares
])
```

## 配置

配置文件位于 `config/otel.ts`：

```ts
// title: config/otel.ts
import { defineConfig } from '@adonisjs/otel'
import env from '#start/env'

export default defineConfig({
  serviceName: env.get('APP_NAME'),
  serviceVersion: env.get('APP_VERSION'),
  environment: env.get('APP_ENV'),
})
```

### 服务标识

该包从多个来源解析服务元数据：

| 选项             | 环境变量                          | 默认值            |
| ---------------- | --------------------------------- | ----------------- |
| `serviceName`    | `OTEL_SERVICE_NAME` 或 `APP_NAME` | `unknown_service` |
| `serviceVersion` | `APP_VERSION`                     | `0.0.0`           |
| `environment`    | `APP_ENV`                         | `development`     |

### 导出器

默认情况下，该包使用 OTLP over gRPC 将追踪数据导出到 `localhost:4317`。这是标准的 OpenTelemetry Collector 端点。如果你在本地或基础设施中运行 OpenTelemetry Collector，追踪数据将自动发送到那里。

你可以使用环境变量配置导出器端点，无需更改任何代码：

```dotenv
OTEL_EXPORTER_OTLP_ENDPOINT=https://otel-collector.example.com:4317
```

对于身份验证或自定义标头：

```dotenv
OTEL_EXPORTER_OTLP_HEADERS=x-api-key=your-api-key
```

有关所有可用选项，请参阅 [OpenTelemetry 环境变量规范](https://opentelemetry.io/docs/specs/otel/configuration/sdk-environment-variables/)，并查看 [高级配置](#advanced-configuration) 以获取更多自定义选项。

### 调试模式

启用调试模式可在开发过程中将 span 打印到控制台：

```ts
// title: config/otel.ts
import { defineConfig } from '@adonisjs/otel'

export default defineConfig({
  serviceName: 'my-app',
  debug: true,
})
```

这会添加一个 `ConsoleSpanExporter`，将 span 输出到你的终端，帮助你在不设置 collector 的情况下可视化追踪。

### 启用和禁用

当 `NODE_ENV === 'test'` 时，OpenTelemetry 会自动禁用，以避免测试期间的噪音。你可以覆盖此行为：

```ts
// title: config/otel.ts
import { defineConfig } from '@adonisjs/otel'

export default defineConfig({
  serviceName: 'my-app',

  /**
   * 在测试中强制启用
   */
  enabled: true,

  /**
   * 或在任何环境中强制禁用
   */
  enabled: false,
})
```

### 采样

在高流量生产环境中，追踪每个请求会生成大量数据。采样控制收集的追踪百分比：

```ts
// title: config/otel.ts
import { defineConfig } from '@adonisjs/otel'

export default defineConfig({
  serviceName: 'my-app',

  /**
   * 采样 10% 的追踪（推荐用于高流量生产环境）
   */
  samplingRatio: 0.1,

  /**
   * 采样 100% 的追踪（默认值，适用于开发环境）
   */
  samplingRatio: 1.0,
})
```

采样器使用基于父级的采样，这意味着子 span 继承父级的采样决策。这确保你始终获得完整的追踪，而不是片段。

另请参阅：[OpenTelemetry 采样文档](https://opentelemetry.io/docs/concepts/sampling/)

:::note

如果你提供自定义 `sampler` 选项，`samplingRatio` 将被忽略。

:::

## 自动检测

在底层，`@adonisjs/otel` 使用官方的 [`@opentelemetry/auto-instrumentations-node`](https://github.com/open-telemetry/opentelemetry-js-contrib/tree/main/packages/auto-instrumentations-node#readme) 包。这意味着你无需任何代码更改即可对广泛的库进行自动追踪：HTTP 请求（传入和传出）、数据库查询（PostgreSQL、MySQL、MongoDB）、Redis 操作以及[更多](https://github.com/open-telemetry/opentelemetry-js-contrib/tree/main/packages/auto-instrumentations-node#supported-instrumentations)。

我们预先配置了合理的默认值，因此一切都开箱即用。但是，你可以通过配置中的 `instrumentations` 选项覆盖任何检测设置。

### 默认忽略的 URL

为了减少噪音，默认情况下以下端点被排除在追踪之外：

- `/health`、`/healthz`、`/ready`、`/readiness`
- `/metrics`、`/internal/metrics`、`/internal/healthz`
- `/favicon.ico`
- 所有 `OPTIONS` 请求（CORS 预检）

### 自定义检测

你可以配置单个检测或添加自定义忽略的 URL：

```ts
// title: config/otel.ts
import { defineConfig } from '@adonisjs/otel'
import { MyCustomInstrumentation } from 'my-custom-instrumentation'

export default defineConfig({
  serviceName: 'my-app',

  instrumentations: {
    /**
     * 添加自定义忽略的 URL（与默认值合并）
     */
    '@opentelemetry/instrumentation-http': {
      ignoredUrls: ['/internal/*', '/api/ping'],
      mergeIgnoredUrls: true,
    },

    /**
     * 禁用特定检测
     */
    '@opentelemetry/instrumentation-pg': { enabled: false },

    /**
     * 添加自定义检测
     */
    'my-custom-instrumentation': new MyCustomInstrumentation(),
  },
})
```

## 创建自定义 span

虽然自动检测涵盖了大多数常见操作，但你通常希望追踪自定义业务逻辑。该包提供了用于此目的的助手和装饰器。

### 使用 record 助手

`record` 助手围绕代码段创建一个 span：

```ts
// title: app/services/order_service.ts
import { record } from '@adonisjs/otel/helpers'

export default class OrderService {
  async processOrder(orderId: string) {
    /**
     * 将同步或异步代码包装在 span 中
     */
    const result = await record('order.process', async () => {
      const order = await Order.findOrFail(orderId)
      await this.validateInventory(order)
      await this.chargePayment(order)
      return order
    })

    return result
  }

  async validateInventory(order: Order) {
    /**
     * 访问 span 以添加自定义属性
     */
    await record('order.validate_inventory', async (span) => {
      span.setAttributes({
        'order.id': order.id,
        'order.items_count': order.items.length,
      })

      // 验证逻辑...
    })
  }
}
```

### 使用装饰器

对于类方法，装饰器提供了更简洁的语法：

```ts
// title: app/services/user_service.ts
import { span, spanAll } from '@adonisjs/otel/decorators'

export default class UserService {
  /**
   * 创建一个名为 "UserService.findById" 的 span
   */
  @span()
  async findById(id: string) {
    return User.find(id)
  }

  /**
   * 自定义 span 名称和属性
   */
  @span({ name: 'user.create', attributes: { operation: 'create' } })
  async create(data: UserData) {
    return User.create(data)
  }
}
```

要自动追踪类的所有方法，请使用 `@spanAll` 装饰器：

```ts
// title: app/services/payment_service.ts
import { spanAll } from '@adonisjs/otel/decorators'

/**
 * 所有方法都获得 span："PaymentService.charge"、"PaymentService.refund" 等
 */
@spanAll()
export default class PaymentService {
  async charge(amount: number) {
    // ...
  }

  async refund(transactionId: string) {
    // ...
  }
}

/**
 * 自定义前缀："payment.charge"、"payment.refund" 等
 */
@spanAll({ prefix: 'payment' })
export default class PaymentService {
  // ...
}
```

### 在当前 span 上设置属性

使用 `setAttributes` 可以向当前活动的 span 添加元数据，而无需创建新的 span：

```ts
// title: app/controllers/orders_controller.ts
import { setAttributes } from '@adonisjs/otel/helpers'

export default class OrdersController {
  async store({ request }: HttpContext) {
    const data = request.all()

    /**
     * 向 HTTP span 添加业务上下文
     */
    setAttributes({
      'order.type': data.type,
      'order.total': data.total,
      'customer.tier': data.customerTier,
    })

    // 处理订单...
  }
}
```

### 记录事件

事件是 span 内的时间戳注释。使用它们来标记重要时刻：

```ts
// title: app/services/checkout_service.ts
import { recordEvent } from '@adonisjs/otel/helpers'

export default class CheckoutService {
  async process(cart: Cart) {
    recordEvent('checkout.started')

    await this.validateCart(cart)
    recordEvent('checkout.cart_validated')

    const payment = await this.processPayment(cart)
    recordEvent('checkout.payment_processed', {
      'payment.method': payment.method,
      'payment.amount': payment.amount,
    })

    await this.fulfillOrder(cart)
    recordEvent('checkout.completed')
  }
}
```

## 上下文传播

当你的应用调用其他服务或处理后台作业时，你需要传播追踪上下文，以便所有操作都出现在同一个 trace 中。

### 传播到队列作业

在调度后台作业时，包含追踪上下文：

```ts
// title: app/controllers/orders_controller.ts
import { injectTraceContext } from '@adonisjs/otel/helpers'

export default class OrdersController {
  async store({ request }: HttpContext) {
    const order = await Order.create(request.all())

    /**
     * 在作业元数据中包含追踪上下文
     */
    const traceHeaders: Record<string, string> = {}
    injectTraceContext(traceHeaders)

    await queue.dispatch('process-order', {
      orderId: order.id,
      traceContext: traceHeaders,
    })

    return order
  }
}
```

在队列工作器中，提取上下文并继续追踪：

```ts
// title: app/jobs/process_order_job.ts
import { extractTraceContext, record } from '@adonisjs/otel/helpers'
import { context } from '@adonisjs/otel'

export default class ProcessOrderJob {
  async handle(payload: {
    orderId: string
    traceContext: Record<string, string>
  }) {
    /**
     * 从作业负载中提取追踪上下文
     */
    const extractedContext = extractTraceContext(payload.traceContext)

    /**
     * 在提取的上下文中运行作业
     */
    await context.with(extractedContext, async () => {
      await record('job.process_order', async () => {
        /**
         * 这个 span 将是原始 HTTP 请求 span 的子 span
         */
        const order = await Order.findOrFail(payload.orderId)
        await this.fulfillOrder(order)
      })
    })
  }
}
```

## 用户上下文

当安装了 `@adonisjs/auth` 时，如果用户已通过身份验证，中间件会自动在 span 上设置用户属性。这包括 `user.id`、`user.email`（如果可用）和 `user.roles`（如果可用）。

:::note
要使自动用户检测生效，用户必须在 OTEL 中间件运行之前通过身份验证。这意味着你需要在路由器中间件栈中**在** OTEL 中间件**之前**注册 `silent_auth` 中间件。如果此设置不适合你的需求，你可以在自己的身份验证中间件中手动使用 `setUser()`。
:::

你可以自定义此行为或添加其他用户属性：

```ts
// title: config/otel.ts
import { defineConfig } from '@adonisjs/otel'

export default defineConfig({
  serviceName: 'my-app',

  /**
   * 禁用自动用户上下文
   */
  userContext: false,

  /**
   * 或使用解析器自定义
   */
  userContext: {
    resolver: async (ctx) => {
      if (!ctx.auth.user) return null

      return {
        id: ctx.auth.user.id,
        tenantId: ctx.auth.user.tenantId,
        plan: ctx.auth.user.plan,
      }
    },
  },
})
```

你也可以在代码中的任何位置手动设置用户上下文。自定义属性会自动添加 `user.` 前缀：

```ts
// title: app/middleware/auth_middleware.ts
import { setUser } from '@adonisjs/otel/helpers'

export default class AuthMiddleware {
  async handle({ auth }: HttpContext, next: NextFn) {
    await auth.authenticate()

    setUser({
      id: auth.user?.id,
      email: auth.user?.email,
      role: auth.user?.role,
      // 自定义属性
      tenantId: auth.user?.tenantId,
      plan: auth.user?.plan,
    })

    return next()
  }
}
```

:::warning
注意不要在用户属性中包含敏感数据（密码、令牌、API 密钥）。这些值会发送到你的可观测性后端，可能会在追踪查看器中可见。
:::

## 日志集成

该包会自动将追踪上下文注入到 Pino 日志中，为每个日志条目添加 `trace_id` 和 `span_id`。这让你可以在可观测性平台中关联日志和追踪。

在开发环境中使用 `pino-pretty` 时，你可以隐藏这些字段以获得更清晰的输出：

```ts
// title: config/logger.ts
import { defineConfig, targets } from '@adonisjs/core/logger'
import app from '@adonisjs/core/services/app'
import { otelLoggingPreset } from '@adonisjs/otel/helpers'

export default defineConfig({
  default: 'app',
  loggers: {
    app: {
      transport: {
        targets: targets()
          .pushIf(!app.inProduction, targets.pretty({ ...otelLoggingPreset() }))
          .toArray(),
      },
    },
  },
})
```

要保持特定字段可见：

```ts
otelLoggingPreset({ keep: ['trace_id', 'span_id'] })
```

## 高级配置

`defineConfig` 函数接受 [OpenTelemetry Node SDK](https://opentelemetry.io/docs/languages/js/getting-started/nodejs/) 的所有选项，为高级用户提供完全控制：

```ts
// title: config/otel.ts
import { defineConfig } from '@adonisjs/otel'
import { BatchSpanProcessor } from '@opentelemetry/sdk-trace-base'
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http'

export default defineConfig({
  serviceName: 'my-app',

  /**
   * 使用 HTTP 而不是 gRPC
   */
  traceExporter: new OTLPTraceExporter({
    url: 'https://otel-collector.example.com/v1/traces',
    headers: { 'x-api-key': process.env.OTEL_API_KEY },
  }),

  /**
   * 带有批处理配置的自定义 span 处理器
   */
  spanProcessors: [
    new BatchSpanProcessor(new OTLPTraceExporter(), {
      maxQueueSize: 2048,
      scheduledDelayMillis: 5000,
    }),
  ],

  /**
   * 自定义资源属性
   */
  resourceAttributes: {
    'deployment.region': 'eu-west-1',
    'k8s.pod.name': process.env.POD_NAME,
  },
})
```

有关所有可用选项，请参阅 [OpenTelemetry JS 文档](https://opentelemetry.io/docs/languages/js/)。

## 性能考虑

OpenTelemetry 会给你的应用增加一些开销。SDK 需要创建 span 对象、记录时间信息并将数据导出到你的 collector。在大多数应用中，这种开销可以忽略不计，但你应该了解它。

对于高流量生产环境，请考虑以下建议：

- **使用采样** 来减少追踪量。`samplingRatio` 为 `0.1`（10%）通常足以识别问题，同时显著减少开销和存储成本。

- **使用批处理**（默认）而不是立即发送 span。`BatchSpanProcessor` 对 span 进行排队并批量发送，减少网络开销。

- **谨慎使用自定义 span**。自动检测涵盖了大多数需求。仅为需要额外可见性的业务关键操作添加自定义 span。不要过度检测，例如在每个类上使用 `@spanAll` 装饰器。

另请参阅：[OpenTelemetry 采样文档](https://opentelemetry.io/docs/concepts/sampling/)

## 助手参考

所有助手都可从 `@adonisjs/otel/helpers` 获取：

| 助手                             | 描述                                            |
| -------------------------------- | ----------------------------------------------- |
| `getCurrentSpan()`               | 返回当前活动的 span，如果没有则返回 `undefined` |
| `setAttributes(attributes)`      | 在当前 span 上设置属性                          |
| `record(name, fn)`               | 在函数周围创建一个 span                         |
| `recordEvent(name, attributes?)` | 在当前 span 上记录事件                          |
| `setUser(user)`                  | 在当前 span 上设置用户属性                      |
| `injectTraceContext(carrier)`    | 将追踪上下文注入到载体对象（标头）中            |
| `extractTraceContext(carrier)`   | 从载体对象中提取追踪上下文                      |
| `otelLoggingPreset(options?)`    | 返回隐藏 OTEL 字段的 pino-pretty 选项           |
