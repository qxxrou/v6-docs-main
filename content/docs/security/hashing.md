---
summary: Learn how to hash values using the AdonisJS hash service.
---

# 哈希

您可以使用 `hash` 服务在应用程序中哈希用户密码。AdonisJS 对 `bcrypt`、`scrypt` 和 `argon2` 哈希算法提供一流的支持，并能够 [添加自定义驱动程序](#creating-a-custom-hash-driver)。

哈希值以 [PHC 字符串格式](https://github.com/P-H-C/phc-string-format/blob/master/phc-sf-spec.md) 存储。PHC 是一种确定性编码规范，用于格式化哈希。

## 使用

`hash.make` 方法接受一个纯字符串值（用户密码输入）并返回哈希输出。

```ts
import hash from '@adonisjs/core/services/hash'

const hash = await hash.make('user_password')
// $scrypt$n=16384,r=8,p=1$iILKD1gVSx6bqualYqyLBQ$DNzIISdmTQS6sFdQ1tJ3UCZ7Uun4uGHNjj0x8FHOqB0pf2LYsu9Xaj5MFhHg21qBz8l5q/oxpeV+ZkgTAj+OzQ
```

您 [不能将哈希值转换回纯文本](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html#hashing-vs-encryption)，哈希是单向过程，生成哈希后无法检索原始值。

然而，哈希提供了一种方法来验证给定的纯文本值是否与现有哈希匹配，您可以使用 `hash.verify` 方法执行此检查。

```ts
import hash from '@adonisjs/core/services/hash'

if (await hash.verify(existingHash, plainTextValue)) {
  // 密码正确
}
```

## 配置

哈希的配置存储在 `config/hash.ts` 文件中。默认驱动程序设置为 `scrypt`，因为 scrypt 使用 Node.js 原生加密模块，不需要任何第三方包。

```ts
// title: config/hash.ts
import { defineConfig, drivers } from '@adonisjs/core/hash'

export default defineConfig({
  default: 'scrypt',

  list: {
    scrypt: drivers.scrypt(),

    /**
     * Uncomment when using argon2
       argon: drivers.argon2(),
     */

    /**
     * Uncomment when using bcrypt
       bcrypt: drivers.bcrypt(),
     */
  },
})
```

### Argon

Argon 是推荐用于哈希用户密码的哈希算法。要在 AdonisJS 哈希服务中使用 argon，您必须安装 [argon2](https://npmjs.com/argon2) npm 包。

```sh
npm i argon2
```

我们使用安全默认值配置 argon 驱动程序，但请根据您的应用程序要求随意调整配置选项。以下是可用选项的列表。

```ts
export default defineConfig({
  // highlight-start
  // 确保将默认驱动程序更新为 argon
  default: 'argon',
  // highlight-end

  list: {
    argon: drivers.argon2({
      version: 0x13, // hex code for 19
      variant: 'id',
      iterations: 3,
      memory: 65536,
      parallelism: 4,
      saltSize: 16,
      hashLength: 32,
    }),
  },
})
```

<dl>

<dt>

variant

</dt>

<dd>

要使用的 argon 哈希变体。

- `d` 速度更快，对 GPU 攻击具有高度抵抗力，这对加密货币很有用
- `i` 速度较慢，对权衡攻击具有抵抗力，这是密码哈希和密钥派生的首选。
- `id` _(默认)_ 是上述的混合组合，对 GPU 和权衡攻击具有抵抗力。

</dd>

<dt>

version

</dt>

<dd>

要使用的 argon 版本。可用选项是 `0x10 (1.0)` 和 `0x13 (1.3)`。默认情况下应使用最新版本。

</dd>

<dt>

iterations

</dt>

<dd>

`iterations` 计数增加了哈希强度，但计算时间更长。

默认值为 `3`。

</dd>

<dt>

memory

</dt>

<dd>

用于哈希值的内存量。每个并行线程将有一个此大小的内存池。

默认值为 `65536 (KiB)`。

</dd>

<dt>

parallelism

</dt>

<dd>

用于计算哈希的线程数。

默认值为 `4`。

</dd>

<dt>

saltSize

</dt>

<dd>

盐的长度（以字节为单位）。Argon 在计算哈希时会生成此大小的加密安全随机盐。

密码哈希的默认值和推荐值为 `16`。

</dd>

<dt>

hashLength

</dt>

<dd>

原始哈希的最大长度（以字节为单位）。输出值将比提到的哈希长度更长，因为原始哈希输出被进一步编码为 PHC 格式。

默认值为 `32`

</dd>

</dl>

### Bcrypt

要在 AdonisJS 哈希服务中使用 Bcrypt，您必须安装 [bcrypt](http://npmjs.com/bcrypt) npm 包。

```sh
npm i bcrypt
```

以下是可用配置选项的列表。

```ts
export default defineConfig({
  // highlight-start
  // 确保将默认驱动程序更新为 bcrypt
  default: 'bcrypt',
  // highlight-end

  list: {
    bcrypt: drivers.bcrypt({
      rounds: 10,
      saltSize: 16,
      version: 98,
    }),
  },
})
```

<dl>

<dt>

rounds

</dt>

<dd>

计算哈希的成本。我们建议阅读 Bcrypt 文档中的 [关于 Rounds 的说明](https://github.com/kelektiv/node.bcrypt.js#a-note-on-rounds) 部分，以了解 `rounds` 值如何影响计算哈希所需的时间。

默认值为 `10`。

</dd>

<dt>

saltSize

</dt>

<dd>

盐的长度（以字节为单位）。计算哈希时，我们会生成此大小的加密安全随机盐。

默认值为 `16`。

</dd>

<dt>

version

</dt>

<dd>

哈希算法的版本。支持的值为 `97` 和 `98`。建议使用最新版本，即 `98`。

</dd>

</dl>

### Scrypt

scrypt 驱动程序使用 Node.js crypto 模块计算密码哈希。配置选项与 [Node.js `scrypt` 方法](https://nodejs.org/dist/latest-v19.x/docs/api/crypto.html#cryptoscryptpassword-salt-keylen-options-callback) 接受的选项相同。

```ts
export default defineConfig({
  // highlight-start
  // 确保将默认驱动程序更新为 scrypt
  default: 'scrypt',
  // highlight-end

  list: {
    scrypt: drivers.scrypt({
      cost: 16384,
      blockSize: 8,
      parallelization: 1,
      saltSize: 16,
      maxMemory: 33554432,
      keyLength: 64,
    }),
  },
})
```

## 使用模型钩子哈希密码

由于您将使用 `hash` 服务哈希用户密码，您可能会发现将逻辑放在 `beforeSave` 模型钩子中很有帮助。

:::note

如果您使用 `@adonisjs/auth` 模块，则不需要在模型内哈希密码。`AuthFinder` 自动处理密码哈希，确保您的用户凭据得到安全处理。在此处了解有关此过程的更多信息 [此处](../authentication/verifying_user_credentials.md#hashing-user-password)。

:::

```ts
import { BaseModel, beforeSave } from '@adonisjs/lucid'
import hash from '@adonisjs/core/services/hash'

export default class User extends BaseModel {
  @beforeSave()
  static async hashPassword(user: User) {
    if (user.$dirty.password) {
      user.password = await hash.make(user.password)
    }
  }
}
```

## 在驱动程序之间切换

如果您的应用程序使用多个哈希驱动程序，您可以使用 `hash.use` 方法在它们之间切换。

`hash.use` 方法接受配置文件中的映射名称并返回匹配驱动程序的实例。

```ts
import hash from '@adonisjs/core/services/hash'

// 使用配置文件中的 "list.scrypt" 映射
await hash.use('scrypt').make('secret')

// 使用配置文件中的 "list.bcrypt" 映射
await hash.use('bcrypt').make('secret')

// 使用配置文件中的 "list.argon" 映射
await hash.use('argon').make('secret')
```

## 检查密码是否需要重新哈希

最新的配置选项建议保持密码安全，尤其是当较旧版本的哈希算法报告漏洞时。

在使用最新选项更新配置后，您可以使用 `hash.needsReHash` 方法检查密码哈希是否使用旧选项并执行重新哈希。

必须在用户登录期间执行检查，因为这是您可以访问纯文本密码的唯一时间。

```ts
import hash from '@adonisjs/core/services/hash'

if (await hash.needsReHash(user.password)) {
  user.password = await hash.make(plainTextPassword)
  await user.save()
}
```

如果您使用模型钩子计算哈希，您可以将纯文本值分配给 `user.password`。

```ts
if (await hash.needsReHash(user.password)) {
  // 让模型钩子重新哈希密码
  user.password = plainTextPassword
  await user.save()
}
```

## 在测试期间模拟哈希服务

哈希值通常是一个缓慢的过程，它会使您的测试变慢。因此，您可能会考虑使用 `hash.fake` 方法模拟哈希服务以禁用密码哈希。

在以下示例中，我们使用 `UserFactory` 创建 20 个用户。由于您使用模型钩子哈希密码，可能需要 5-7 秒（取决于配置）。

```ts
import hash from '@adonisjs/core/services/hash'

test('获取用户列表', async ({ client }) => {
  await UserFactory().createMany(20)
  const response = await client.get('users')
})
```

然而，一旦您模拟了哈希服务，相同的测试将运行得更快。

```ts
import hash from '@adonisjs/core/services/hash'

test('获取用户列表', async ({ client }) => {
  // highlight-start
  hash.fake()
  // highlight-end

  await UserFactory().createMany(20)
  const response = await client.get('users')

  // highlight-start
  hash.restore()
  // highlight-end
})
```

## 创建自定义哈希驱动程序

哈希驱动程序必须实现 [HashDriverContract](https://github.com/adonisjs/hash/blob/main/src/types.ts#L13) 接口。此外，官方 Hash 驱动程序使用 [PHC 格式](https://github.com/P-H-C/phc-string-format/blob/master/phc-sf-spec.md) 序列化哈希输出以进行存储。您可以检查现有驱动程序的实现，了解它们如何使用 [PHC 格式化程序](https://github.com/adonisjs/hash/blob/main/src/drivers/bcrypt.ts) 来创建和验证哈希。

```ts
import {
  HashDriverContract,
  ManagerDriverFactory,
} from '@adonisjs/core/types/hash'

/**
 * 哈希驱动程序接受的配置
 */
export type PbkdfConfig = {}

/**
 * 驱动程序实现
 */
export class Pbkdf2Driver implements HashDriverContract {
  constructor(public config: PbkdfConfig) {}

  /**
   * 检查哈希值是否按照哈希算法的格式格式化。
   */
  isValidHash(value: string): boolean {}

  /**
   * 将原始值转换为哈希
   */
  async make(value: string): Promise<string> {}

  /**
   * 验证纯文本值是否与提供的哈希匹配
   */
  async verify(hashedValue: string, plainValue: string): Promise<boolean> {}

  /**
   * 检查哈希是否需要重新哈希，因为配置参数已更改
   */
  needsReHash(value: string): boolean {}
}

/**
 * 工厂函数，用于在配置文件中引用驱动程序。
 */
export function pbkdf2Driver(config: PbkdfConfig): ManagerDriverFactory {
  return () => {
    return new Pbkdf2Driver(config)
  }
}
```

在上面的代码示例中，我们导出以下值。

- `PbkdfConfig`：您想要接受的配置的 TypeScript 类型。

- `Pbkdf2Driver`：驱动程序的实现。它必须遵循 `HashDriverContract` 接口。

- `pbkdf2Driver`：最后，一个工厂函数来延迟创建驱动程序的实例。

### 使用驱动程序

实现完成后，您可以使用 `pbkdf2Driver` 工厂函数在配置文件中引用驱动程序。

```ts
// title: config/hash.ts
import { defineConfig } from '@adonisjs/core/hash'
import { pbkdf2Driver } from 'my-custom-package'

export default defineConfig({
  list: {
    pbkdf2: pbkdf2Driver({
      // 配置放在这里
    }),
  },
})
```
