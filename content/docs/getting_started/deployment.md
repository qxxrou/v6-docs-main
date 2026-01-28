---
summary: 了解在生产环境中部署 AdonisJS 应用程序的一般准则。
---

# 部署

部署 AdonisJS 应用程序与部署标准 Node.js 应用程序没有什么不同。你需要一个运行 `Node.js >= 20.6` 的服务器以及 `npm` 来安装生产依赖项。

本指南将介绍在生产环境中部署和运行 AdonisJS 应用程序的一般准则。

## 创建生产构建

作为第一步，你必须通过运行 `build` 命令创建 AdonisJS 应用程序的生产构建。

另请参阅：[TypeScript 构建过程](../concepts/typescript_build_process.md)

```sh
node ace build
```

编译后的输出写入 `./build` 目录。如果你使用 Vite，其输出将写入 `./build/public` 目录。

创建生产构建后，你可以将 `./build` 文件夹复制到你的生产服务器。**从现在开始，构建文件夹将是你的应用程序的根目录**。

### 创建 Docker 镜像

如果你正在使用 Docker 部署你的应用程序，你可以使用以下 `Dockerfile` 创建 Docker 镜像。

```dockerfile
FROM node:22.16.0-alpine3.22 AS base

# 所有依赖项阶段
FROM base AS deps
WORKDIR /app
ADD package.json package-lock.json ./
RUN npm ci

# 仅生产依赖项阶段
FROM base AS production-deps
WORKDIR /app
ADD package.json package-lock.json ./
RUN npm ci --omit=dev

# 构建阶段
FROM base AS build
WORKDIR /app
COPY --from=deps /app/node_modules /app/node_modules
ADD . .
RUN node ace build

# 生产阶段
FROM base
ENV NODE_ENV=production
WORKDIR /app
COPY --from=production-deps /app/node_modules /app/node_modules
COPY --from=build /app/build /app
EXPOSE 8080
CMD ["node", "./bin/server.js"]
```

请随意修改 Dockerfile 以满足你的需求。

## 配置反向代理

Node.js 应用程序通常[部署在反向代理](https://medium.com/intrinsic-blog/why-should-i-use-a-reverse-proxy-if-node-js-is-production-ready-5a079408b2ca)服务器（如 Nginx）后面。因此，端口 `80` 和 `443` 上的传入流量将首先由 Nginx 处理，然后转发到你的 Node.js 应用程序。

以下是你可用作起点的示例 Nginx 配置文件。

:::warning
确保替换尖括号 `<>` 内的值。
:::

```nginx
server {
  listen 80;
  listen [::]:80;

  server_name <APP_DOMAIN.COM>;

  location / {
    proxy_pass http://localhost:<ADONIS_PORT>;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_cache_bypass $http_upgrade;
  }
}
```

## 定义环境变量

如果你在裸机服务器（如 DigitalOcean Droplet 或 EC2 实例）上部署你的应用程序，你可以使用 `.env` 文件来定义环境变量。确保文件安全存储，只有授权用户才能访问它。

:::note
如果你正在使用 Heroku 或 Cleavr 等部署平台，你可以使用他们的控制面板来定义环境变量。
:::

假设你已在 `/etc/secrets` 目录中创建了 `.env` 文件，你必须按如下方式启动你的生产服务器。

```sh
ENV_PATH=/etc/secrets node build/bin/server.js
```

`ENV_PATH` 环境变量指示 AdonisJS 在提到的目录中查找 `.env` 文件。

## 启动生产服务器

你可以通过运行 `node server.js` 文件来启动生产服务器。然而，建议使用像 [pm2](https://pm2.keymetrics.io/docs/usage/quick-start) 这样的进程管理器。

- PM2 将在后台运行你的应用程序，而不会阻塞当前的终端会话。
- 如果你的应用在服务请求时崩溃，它将重新启动应用程序。
- 此外，PM2 使得在[集群模式](https://nodejs.org/api/cluster.html#cluster)下运行你的应用程序变得超级简单。

以下是你可用作起点的示例 [pm2 生态系统文件](https://pm2.keymetrics.io/docs/usage/application-declaration)。

```js
// title: ecosystem.config.js
module.exports = {
  apps: [
    {
      name: 'web-app',
      script: './server.js',
      instances: 'max',
      exec_mode: 'cluster',
      autorestart: true,
    },
  ],
}
```

```sh
// title: 启动服务器
pm2 start ecosystem.config.js
```

## 迁移数据库

如果你使用的是 SQL 数据库，你必须在生产服务器上运行数据库迁移以创建所需的表。

如果你使用的是 Lucid，你可以运行以下命令。

```sh
node ace migration:run --force
```

在生产环境中运行迁移时需要 `--force` 标志。

### 何时运行迁移

此外，你应始终在重新启动服务器之前运行迁移。然后，如果迁移失败，请不要重新启动服务器。

使用 Cleavr 或 Heroku 等托管服务，他们可以自动处理此用例。否则，你将不得不在 CI/CD 管道内运行迁移脚本或通过 SSH 手动运行它。

### 不要在生产中回滚

在生产环境中回滚迁移是一项有风险的操作。迁移文件中的 `down` 方法通常包含破坏性操作，如**删除表**或**删除列**等。

建议在生产环境中的 config/database.ts 文件内关闭回滚，而是创建一个新的迁移来修复问题并在生产服务器上运行它。

在生产环境中禁用回滚将确保 `node ace migration:rollback` 命令导致错误。

```js
{
  pg: {
    client: 'pg',
    migrations: {
      disableRollbacksInProduction: true,
    }
  }
}
```

### 并发迁移

如果你在具有多个实例的服务器上运行迁移，你必须确保只有一个实例运行迁移。

对于 MySQL 和 PostgreSQL，Lucid 将获得咨询锁以确保不允许并发迁移。然而，最好首先避免从多个服务器运行迁移。

## 文件上传的持久存储

Amazon EKS、Google Kubernetes、Heroku、DigitalOcean Apps 等环境在[临时文件系统](https://devcenter.heroku.com/articles/dynos#ephemeral-filesystem)内运行你的应用程序代码，这意味着每次部署默认情况下都会销毁现有文件系统并创建一个新的文件系统。

如果你的应用程序允许用户上传文件，你必须使用像 Amazon S3、Google Cloud Storage 或 DigitalOcean Spaces 这样的持久存储服务，而不是依赖于本地文件系统。

## 写入日志

AdonisJS 默认使用 [`pino` 日志记录器](../digging_deeper/logger.md)，它以 JSON 格式将日志写入控制台。你可以设置外部日志记录服务以从 stdout/stderr 读取日志，或将它们转发到同一服务器上的本地文件。

## 提供静态资源

有效地提供静态资源对于应用程序的性能至关重要。无论你的 AdonisJS 应用程序有多快，静态资源的交付在更好的用户体验中都起着重要作用。

### 使用 CDN

最佳方法是使用 CDN（内容分发网络）来交付来自 AdonisJS 应用程序的静态资源。

使用 [Vite](../basics/vite.md) 编译的前端资源默认是指纹化的，这意味着文件名基于其内容进行哈希处理。这允许你永久缓存资源并从 CDN 提供它们。

根据你使用的 CDN 服务和部署技术，你可能必须在部署过程中添加一个步骤，以将静态文件移动到 CDN 服务器。这就是它应该如何在 nutshell 中工作。

1. 更新 `vite.config.js` 和 `config/vite.ts` 配置以[使用 CDN URL](../basics/vite.md#deploying-assets-to-a-cdn)。

2. 运行 `build` 命令来编译应用程序和资源。

3. 将 `public/assets` 的输出复制到你的 CDN 服务器。例如，[这是我们在](https://github.com/adonisjs-community/polls-app/blob/main/commands/PublishAssets.ts)用于将资源发布到 Amazon S3 存储桶的命令。

### 使用 Nginx 交付静态资源

另一个选择是将提供资源的任务卸载给 Nginx。如果你使用 Vite 编译前端资源，由于它们已被指纹化，你必须积极地缓存所有静态文件。

将以下块添加到你的 Nginx 配置文件中。**确保替换尖括号 `<>` 内的值**。

```nginx
location ~ \.(jpg|png|css|js|gif|ico|woff|woff2) {
  root <PATH_TO_ADONISJS_APP_PUBLIC_DIRECTORY>;
  sendfile on;
  sendfile_max_chunk 2m;
  add_header Cache-Control "public";
  expires 365d;
}
```

### 使用 AdonisJS 静态文件服务器

你也可以依赖 [AdonisJS 内置静态文件服务器](../basics/static_file_server.md)来提供来自应用程序 `public` 目录的静态资源，以保持简单。

无需额外配置。只需像往常一样部署你的 AdonisJS 应用程序，对静态资源的请求将自动被提供。

:::warning
不建议在生产环境中使用静态文件服务器。最好使用 CDN 或 Nginx 来提供静态资源。
:::
