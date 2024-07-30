---
title: Outline部署教程
tags: []
categories: []
abbrlink: 9c50151f
date: 2024-01-08 11:28:22
---

# 摘要

因为日常需要写一些自己的学习笔记，文章等，因此需要一个软件用于记录和展示。市面上有很多可用的产品，比如notion，语雀，飞书等等，还有相当一部分可以私有化部署的，比如obsidian，思源笔记，typora等等都可以本地存储。这些唯一的问题是跨设备，需要全量同步数据，对我来说不是很适合。

## 期望的软件

- **纯Web应用**。设备只需要一个浏览器就可以阅读并编辑所以内容
- **可私有化部署**。作为程序员，数据还是掌握在自己手中比较好
- **占用资源小**。因为是跑在CPU及内存都惨不忍睹的NAS中的服务，所以尽量占用资源小
- **支持分享**。偶尔会把文章分享给朋友及同事，能不登陆查看文章。

综合以上期望，最后发现一个国内用户比较少的outline。https://www.getoutline.com/

# 介绍

在其github上，第一句话是

> *使用 React 和 Node.js 为您的团队构建的快速、协作、知识库。*

其目标是协作的知识库，官网介绍中也是着重介绍协作的功能。使用前端技术栈编写，界面简洁，提供SASS版本，灭月最少10刀。我选择私有化部署

查看其部署文档，我只能说有，但不多。那就抛弃官网，自己摸索吧。

## 依赖

根据其提供的env配置文件，大概需要以下组件：

- PostgreSQL(v12+)
- Redis(v4+)
- S3存储（非必须）
- 身份认证工具

外国佬的东西，全程手动，什么都要自己准备。

## 选型

**附件存储**：outline提供了两种选择，类S3存储，或本地存储。这里选择minio

**身份认证程序**：outline没有提供登录模块，需要集成第三方的登录。我这里选择使用Gitea自带的Oauth2

------

接下来按照步骤一点一点部署，所有组件都使用docker-compose部署

## 域名规划

这里只表述前缀

- **s3**：S3对象存储
- **s3-admin**：Minio控制台
- **gitea**：Gitea
- **outline**：outline的主服务

## 加速访问

因为一般都是内网访问，没必要走一圈公网，所以这里在路由器配置解析，直接把域名解析到NAS。由NAS的Nginx提供服务

# PostgreSQL&Redis&Minio

这里就直接提供docker-compose的文件。

**注意事项**

- PostgreSQL需要提前创建数据库
- Minio需要创建一个匿名访问，用于用户头像的读取

## PostgreSQL

因为文件读写权限的问题，且都是内网访问，这里用privileged开放了docker的用户权限，实际生产不建议使用

```yaml
version: '3'
services:
  postgres:
    image: postgres:13-alpine
    container_name: postgres-13
    ports:
     - 5432:5432
    environment:
      - POSTGRES_PASSWORD=password # feel free to change the password
    volumes:
      - ./postgresql/data:/var/lib/postgresql/data # persist postgres data to ~/postgres/data/ on the host
    privileged: true
    cpus: 2
    mem_limit: 512m
```

## Redis

```yaml
version: '3'
services:
  redis:
    image: redis:4.0.1
    container_name: redis
    volumes:
      - ./datadir:/data
      - ./conf/redis.conf:/usr/local/etc/redis/redis.conf
      - ./logs:/logs
    command: redis-server --requirepass password
    ports:
      - 6379:6379
```

## Minio

```yaml
version: "3.7"
services:
  minio:
    image: "quay.io/minio/minio:RELEASE.2024-01-01T16-36-33Z"
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - "./minio/data1:/data1"
      - "./minio/data2:/data2"
      - "./minio:/data"
    command: server --console-address ":9001" http://minio/data{1...2}
    environment:
      - MINIO_ROOT_USER=admin
      - MINIO_ROOT_PASSWORD=password
```

进入控制台，创建Bucket，AccessKey

**需要添加一个匿名访问策略，用于头像访问**

 ![](https://static.lianglianglee.com/2024/01/b8c1dee5eb74cbbfb864e2e2f196891f.png)

# 身份认证程序

身份认证程序使用的Gitea

```yaml
version: "3"

services:
  server:
    image: gitea/gitea:1.19.3
    container_name: gitea
    environment:
      - GITEA__database__DB_TYPE=mysql
      - GITEA__database__HOST=xxx.xxx.xxx.xxx:3306
      - GITEA__database__NAME=gitea
      - GITEA__database__USER=root
      - GITEA__database__PASSWD=password
    restart: always
    volumes:
      - ./gitea:/data
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "3002:3000"
      - "222:22"
```

gitea使用了MySQL作为数据存储服务

## 创建Oauth2应用

以管理员账号登录gitea后，进入管理后台，应用tab页

![](https://static.lianglianglee.com/2024/01/49d62f29b251c7fa0aab8e62a1b752e6.png)

重定向URL是outline的地址，这里替换成outline的域名或者IP:端口

> http://xxx.xxx.xxx/auth/oidc.callback

创建完成后会生成一堆客户端ID和密钥

![](https://static.lianglianglee.com/2024/01/ccdf3707165be5f883d7c8cbc90faa67.png)

客户端ID和密钥先找个地方记下。

# Outline部署

Outline因为太多的配置项，这里使用env文件处理，官方的env文件链接如下

https://github.com/outline/outline/blob/main/.env.sample

## 配置

env文件分为几大类

- 应用的加密配置
- PostgreSQL，Redis配置
- 应用的域名及端口配置
- 存储配置
- 身份认证配置
- 邮箱配置
- 其他

```ini
# –––––––––––––––– REQUIRED ––––––––––––––––

NODE_ENV=production

# Generate a hex-encoded 32-byte random key. You should use `openssl rand -hex 32`
# in your terminal to generate a random value.
SECRET_KEY=d1020471380935f0053864173d8178578910ab9331a434aa1e4ec20ae6a8bb47

# Generate a unique random key. The format is not important but you could still use
# `openssl rand -hex 32` in your terminal to produce this.
UTILS_SECRET=d1020471380935f0053864173d8178578910ab9331a434aa1e4ec20ae6a8bb47

# 数据库配置
DATABASE_URL=postgres://postgres:password@xxx.xxx.xxx:5432/outline
DATABASE_URL_TEST=postgres://postgres:password@xxx.xxx.xxx:5432/outline-test
DATABASE_CONNECTION_POOL_MIN=
DATABASE_CONNECTION_POOL_MAX=
# 关闭SSL模式
PGSSLMODE=disable

# Redis配置
REDIS_URL=redis://:ll.941107@xxx.xxx.xxx:6379

# Outline的域名及端口
URL=http://outline.xxxx.com
PORT=3004


# S3配置
AWS_ACCESS_KEY_ID=YfIfejBwr2ckKJ4W9Dew
AWS_SECRET_ACCESS_KEY=mqpSPul28PoRs7T3TyEABUTT9zbz1ctcvT277qg1
AWS_REGION=cn-homelab
# AWS_S3_ACCELERATE_URL=
AWS_S3_UPLOAD_BUCKET_URL=http://s3.xxxx.com
AWS_S3_UPLOAD_BUCKET_NAME=outline-blob
AWS_S3_FORCE_PATH_STYLE=true
AWS_S3_ACL=private

# 附件存储方式：S3或者local
FILE_STORAGE=s3

# 身份认证服务的各种URL
# Redirect URI is https://<URL>/auth/oidc.callback
OIDC_CLIENT_ID=c49f442d-5c63-4394-8a56-78256c502b04
OIDC_CLIENT_SECRET=gto_v5ygxr3ym4a2w4tm3vjwebjr2tokr7jwjoob2zmh7swsq6h6jkda
OIDC_AUTH_URI=http://gitea.xxxx.com/login/oauth/authorize
OIDC_TOKEN_URI=http://gitea.xxxx.com/login/oauth/access_token
OIDC_USERINFO_URI=http://gitea.xxxx.com/login/oauth/userinfo

# Specify which claims to derive user information from
# Supports any valid JSON path with the JWT payload
OIDC_USERNAME_CLAIM=name

# Display name for OIDC authentication
OIDC_DISPLAY_NAME=Gitea

# Space separated auth scopes.
OIDC_SCOPES=openid email name


# –––––––––––––––– OPTIONAL ––––––––––––––––

# If using a Cloudfront/Cloudflare distribution or similar it can be set below.
# This will cause paths to javascript, stylesheets, and images to be updated to
# the hostname defined in CDN_URL. In your CDN configuration the origin server
# should be set to the same as URL.
# CDN_URL=

# Auto-redirect to https in production. The default is true but you may set to
# false if you can be sure that SSL is terminated at an external loadbalancer.
FORCE_HTTPS=false

# Have the installation check for updates by sending anonymized statistics to
# the maintainers
ENABLE_UPDATES=false

# 应用启动几个进程，因为只有我一个人访问，设置1即可。按照内存/512M设置数量
WEB_CONCURRENCY=1

# 最大导入大小
MAXIMUM_IMPORT_SIZE=5120000

# You can remove this line if your reverse proxy already logs incoming http
# requests and this ends up being duplicative
DEBUG=http

# Configure lowest severity level for server logs. Should be one of
# error, warn, info, http, verbose, debug and silly
LOG_LEVEL=debug


# 邮箱的配置
SMTP_HOST=smtp.sina.com
SMTP_PORT=465
SMTP_USERNAME=xxxx@sina.com
SMTP_PASSWORD=xxxxx
SMTP_FROM_EMAIL=xxxx@sina.com
SMTP_REPLY_EMAIL=xxxx@sina.com
# SMTP_TLS_CIPHERS=
SMTP_SECURE=true

# 默认语言，这里选择中文
DEFAULT_LANGUAGE=zh_CN

# Optionally enable rate limiter at application web server
RATE_LIMITER_ENABLED=true

# Configure default throttling parameters for rate limiter
RATE_LIMITER_REQUESTS=1000
RATE_LIMITER_DURATION_WINDOW=60


# Enable unsafe-inline in script-src CSP directive
# Setting it to true allows React dev tools add-on in
# Firefox to successfully detect the project
DEVELOPMENT_UNSAFE_INLINE_CSP=false
```

如果您使用的身份认证程序不一样，则按照其文档的URL填写以下参数

- OIDC_AUTH_URI：认证接入点
- OIDC_TOKEN_URI：访问token接入点
- OIDC_USERINFO_URI：用户详情
- OIDC_USERNAME_CLAIM：简单来说用什么作为用户名，从OIDC_SCOPES选择
- OIDC_SCOPES：申请的权限，要根据身份认证程序确定有哪些权限

## 部署

```ini
version: "3.2"
services:

  outline:
    image: docker.getoutline.com/outlinewiki/outline:0.74.0
    env_file: .env
    ports:
      - "3004:3004"
    volumes:
      - ./data:/var/lib/outline/data
```

注意：

- .env就是上面说的配置文件，需要跟docker-compose.yml同目录
- 容器内的3004是env文件的PORT配置

启动完成后，稍等片刻，即可访问http://ip:3004

# 使用手册

https://docs.getoutline.com/s/guide