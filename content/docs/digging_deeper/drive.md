---
summary: 在本地文件系统和云存储服务（如S3、GCS、R2和DigitalOcean Spaces）上管理用户上传的文件，没有任何供应商锁定。
---

# Drive

AdonisJS Drive (`@adonisjs/drive`) 是 [flydrive.dev](https://flydrive.dev/) 之上的轻量级包装器。FlyDrive 是一个用于 Node.js 的文件存储库。它提供了一个统一的 API 来与本地文件系统和云存储解决方案（如 S3、R2 和 GCS）进行交互。

使用 FlyDrive，您可以在各种云存储服务（包括本地文件系统）上管理用户上传的文件，而无需更改一行代码。

## 安装

使用以下命令安装并配置 `@adonisjs/drive` 包：

```sh
node ace add @adonisjs/drive
```

:::disclosure{title="查看 add 命令执行的步骤"}

1. 使用检测到的包管理器安装 `@adonisjs/drive` 包。

2. 在 `adonisrc.ts` 文件中注册以下服务提供者。

   ```ts
   {
     providers: [
       // ...其他提供者
       () => import('@adonisjs/drive/drive_provider'),
     ]
   }
   ```

3. 创建 `config/drive.ts` 文件。

4. 为所选存储服务定义环境变量。

5. 为所选存储服务安装所需的对等依赖项。

:::

## 配置

`@adonisjs/drive` 包的配置存储在 `config/drive.ts` 文件中。您可以在单个配置文件中为多个服务定义配置。

另请参阅：[配置存根](https://github.com/adonisjs/drive/blob/3.x/stubs/config/drive.stub)

```ts
import env from '#start/env'
import app from '@adonisjs/core/services/app'
import { defineConfig, services } from '@adonisjs/drive'

const driveConfig = defineConfig({
  default: env.get('DRIVE_DISK'),

  services: {
    /**
     * 在本地文件系统上持久化文件
     */
    fs: services.fs({
      location: app.makePath('storage'),
      serveFiles: true,
      routeBasePath: '/uploads',
      visibility: 'public',
    }),

    /**
     * 在 Digital Ocean Spaces 上持久化文件
     */
    spaces: services.s3({
      credentials: {
        accessKeyId: env.get('SPACES_KEY'),
        secretAccessKey: env.get('SPACES_SECRET'),
      },
      region: env.get('SPACES_REGION'),
      bucket: env.get('SPACES_BUCKET'),
      endpoint: env.get('SPACES_ENDPOINT'),
      visibility: 'public',
    }),
  },
})

export default driveConfig
```

### 环境变量

存储服务的凭据/设置作为环境变量存储在 `.env` 文件中。在使用 Drive 之前，请确保更新这些值。

此外，`DRIVE_DISK` 环境变量定义了用于管理文件的默认磁盘/服务。例如，您可能希望在开发中使用 `fs` 磁盘，在生产中使用 `spaces` 磁盘。

## 使用

配置好 Drive 后，您可以导入 `drive` 服务来与其 API 进行交互。在以下示例中，我们使用 Drive 处理文件上传操作。

:::note

由于 AdonisJS 集成是 FlyDrive 之上的薄包装器，您应该阅读 [FlyDrive 文档](https://flydrive.dev) 以更好地理解其 API。

:::

```ts
import { cuid } from '@adonisjs/core/helpers'
import drive from '@adonisjs/drive/services/main'
import router from '@adonisjs/core/services/router'

router.put('/me', async ({ request, response }) => {
  /**
   * 步骤 1：从请求中获取图像并执行基本
   * 验证
   */
  const image = request.file('avatar', {
    size: '2mb',
    extnames: ['jpeg', 'jpg', 'png'],
  })
  if (!image) {
    return response.badRequest({ error: '图像缺失' })
  }

  /**
   * 步骤 2：使用 Drive 将图像移动到具有唯一名称的位置
   */
  const key = `uploads/${cuid()}.${image.extname}`
  // highlight-start
  await image.moveToDisk(key)
  // highlight-end

  /**
   * 使用文件的公共 URL 响应
   */
  return {
    message: '图像已上传',
    // highlight-start
    url: image.meta.url,
    // highlight-end
  }
})
```

- Drive 包将 `moveToDisk` 方法添加到 [MultipartFile](https://github.com/adonisjs/drive/blob/develop/providers/drive_provider.ts#L110)。此方法将文件从其 `tmpPath` 移动到配置的存储提供程序。

- 移动文件后，`meta.url` 属性将设置在文件对象上。此属性包含文件的公共 URL。如果您的文件是私有的，则必须使用 `drive.use().getSignedUrl()` 方法。

## Drive 服务

从 `@adonisjs/drive/services/main` 路径导出的 Drive 服务是 [DriveManager](https://flydrive.dev/docs/drive_manager) 类的单例实例，使用从 `config/drive.ts` 文件导出的配置创建。

您可以导入此服务来与 DriveManager 和配置的文件存储服务进行交互。例如：

```ts
import drive from '@adonisjs/drive/services/main'

drive instanceof DriveManager // true

/**
 * 返回默认磁盘的实例
 */
const disk = drive.use()

/**
 * 返回名为 r2 的磁盘实例
 */
const disk = drive.use('r2')

/**
 * 返回名为 spaces 的磁盘实例
 */
const disk = drive.use('spaces')
```

一旦您可以访问 Disk 的实例，您就可以使用它来管理文件。

另请参阅：[Disk API](https://flydrive.dev/docs/disk_api)

```ts
await disk.put(key, value)
await disk.putStream(key, readableStream)

await disk.get(key)
await disk.getStream(key)
await disk.getArrayBuffer(key)

await disk.delete(key)
await disk.deleteAll(prefix)

await disk.copy(source, destination)
await disk.move(source, destination)

await disk.copyFromFs(source, destination)
await disk.moveFromFs(source, destination)
```

## 本地文件系统驱动程序

AdonisJS 集成增强了 FlyDrive 的本地文件系统驱动程序，并添加了对 URL 生成和使用 AdonisJS HTTP 服务器提供文件的支持。

以下是您可以用来配置文件系统驱动程序的选项列表。

```ts
{
  services: {
    fs: services.fs({
      location: app.makePath('storage'),
      visibility: 'public',

      appUrl: env.get('APP_URL'),
      serveFiles: true,
      routeBasePath: '/uploads',
    }),
  }
}
```

<dl>

<dt>

location

</dt>

<dd>

`location` 属性定义了应该存储文件的存储位置。此目录应添加到 `.gitignore` 中，以便您不会将开发期间上传的文件推送到生产服务器。

</dd>

<dt>

visibility

</dt>

<dd>

`visibility` 属性用于将文件标记为公共或私有。私有文件只能使用签名 URL 访问。[了解更多](https://flydrive.dev/docs/disk_api#getsignedurl)

</dd>

<dt>

serveFiles

</dt>

<dd>

`serveFiles` 选项自动注册一条路由，用于从本地文件系统提供文件。您可以使用 [list:routes](../references/commands.md#listroutes) ace 命令查看此路由。

</dd>

<dt>

routeBasePath

</dt>

<dd>

`routeBasePath` 选项定义了用于提供文件的路由的基础前缀。确保基础前缀是唯一的。

</dd>

<dt>

appUrl

</dt>

<dd>

您可以选择定义 `appUrl` 属性，以便创建包含应用程序完整域名的 URL。否则将创建相对 URL。

</dd>

</dl>

## Edge 助手

在 Edge 模板中，您可以使用以下助手方法之一来生成 URL。这两种方法都是异步的，因此请确保 `await` 它们。

```edge
<img src="{{ await driveUrl(user.avatar) }}" />

<!-- 为命名磁盘生成 URL -->
<img src="{{ await driveUrl(user.avatar, 's3') }}" />
<img src="{{ await driveUrl(user.avatar, 'r2') }}" />
```

```edge
<a href="{{ await driveSignedUrl(invoice.key) }}">
  下载发票
</a>

<!-- 为命名磁盘生成 URL -->
<a href="{{ await driveSignedUrl(invoice.key, 's3') }}">
  下载发票
</a>

<!-- 使用签名选项生成 URL -->
<a href="{{ await driveSignedUrl(invoice.key, {
  expiresIn: '30 mins',
}) }}">
  下载发票
</a>
```

## MultipartFile 助手

Drive 扩展了 Bodyparser [MultipartFile](https://github.com/adonisjs/drive/blob/develop/providers/drive_provider.ts#L110) 类并添加了 `moveToDisk` 方法。此方法将文件从其 `tmpPath` 复制到配置的存储提供程序。

```ts
const image = request.file('image')!

const key = 'user-1-avatar.png'

/**
 * 将文件移动到默认磁盘
 */
await image.moveToDisk(key)

/**
 * 将文件移动到命名磁盘
 */
await image.moveToDisk(key, 's3')

/**
 * 在移动操作期间定义其他属性
 */
await image.moveToDisk(key, 's3', {
  contentType: 'image/png',
})

/**
 * 通过先将文件作为缓冲区读取来写入文件。当您的云存储提供程序使用 "stream" 选项导致文件损坏时，您可以使用此选项
 */
await image.moveToDisk(key, 's3', {
  moveAs: 'buffer',
})
```

## 在测试期间模拟磁盘

Drive 的 fakes API 可用于在测试期间防止与远程存储交互。在 fakes 模式下，`drive.use()` 方法将返回一个假磁盘（由本地文件系统支持），所有文件都将写入应用程序根目录的 `./tmp/drive-fakes` 目录中。

这些文件在您使用 `drive.restore` 方法恢复 fake 后会自动删除。

另请参阅：[FlyDrive fakes 文档](https://flydrive.dev/docs/drive_manager#using-fakes)

```ts
// title: tests/functional/users/update.spec.ts
import { test } from '@japa/runner'
import drive from '@adonisjs/drive/services/main'
import fileGenerator from '@poppinss/file-generator'

test.group('Users | update', () => {
  test('should be able to update my avatar', async ({ client, cleanup }) => {
    /**
     * 模拟 "spaces" 磁盘并在测试完成后恢复 fake
     */
    const fakeDisk = drive.fake('spaces')
    cleanup(() => drive.restore('spaces'))

    /**
     * 创建用户以执行登录和更新
     */
    const user = await UserFactory.create()

    /**
     * 生成大小为 1mb 的假内存 png 文件
     */
    const { contents, mime, name } = await fileGenerator.generatePng('1mb')

    /**
     * 发出 put 请求并发送文件
     */
    await client
      .put('me')
      .file('avatar', contents, {
        filename: name,
        contentType: mime,
      })
      .loginAs(user)

    /**
     * 断言文件存在
     */
    fakeDisk.assertExists(user.avatar)
  })
})
```
