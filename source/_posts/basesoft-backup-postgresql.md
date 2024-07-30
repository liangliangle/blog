---
title: 【发现好项目】给你的Postgres加个备份
tags: []
categories: []
abbrlink: fa974d33
date: 2024-07-29 23:03:04
---

GitHub地址：<https://github.com/eduardolat/pgbackweb>

# 项目简介

使用用户友好的 Web 界面轻松备份 PostgreSQL！

特点：


1. 界面友好简介
2. 备份上传到类S3存储
3. 支持定时备份
4. Web页面可直接下载备份
5. 敏感配置加密保存

缺点：


1. 不支持一键备份整个Server，多个数据库需要多次配置
2. 连接字符串没有拆分，导致配置时需要注意格式

# 部署

```yaml
services:
  pgbackweb:
    image: eduardolat/pgbackweb:latest
    ports:
      - "8085:8085" # Access the web interface at http://localhost:8085
    environment:
      PBW_ENCRYPTION_KEY: "my_secret_key"
      PBW_POSTGRES_CONN_STRING: "postgresql://postgres:password@postgres:5432/pgbackweb?sslmode=disable"
    depends_on:
      postgres:
        condition: service_healthy

  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: postgres
      POSTGRES_DB: pgbackweb
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"
    volumes:
      - ./data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
```

替换掉环境变量：PBW_ENCRYPTION_KEY 为任意字符串

如果变更POSTGRES_PASSWORD，记得 PBW_POSTGRES_CONN_STRING 中的password也要变更

## 使用Minio作为备份后的存储

```yaml
version: "3.7"
services:
  minio:
    image: quay.io/minio/minio:RELEASE.2024-01-01T16-36-33Z
    ports:
      - 9000:9000
      - 9001:9001
    volumes:
      - ./minio/minio:/data
    command: server --console-address ":9001" /data
    environment:
      - MINIO_ROOT_USER=AKIAIOSFODNN7EXAMPLE // 请自行变更 
      - MINIO_ROOT_PASSWORD=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY  //请自行变更
```

浏览器打开 ip:9001

输入MINIO_ROOT_USER，MINIO_ROOT_PASSWORD配置的账号密码

 ![Minio管理页](https://static.lianglianglee.com/2024/07/IM7MxgK.png)

 Create a Bucket

 ![create a bucket](https://static.lianglianglee.com/2024/07/KCYMxgK.png)

创建完成后，minio即配置完成

# 使用

整个功能分为三大块

* Database 配置需要备份的数据库链接信息
* Destinations 配置S3存储，备份后的数据上传到该存储中
* Backups 备份的配置，什么时候备份，备份保存天数等

## 创建Database

 ![](https://static.lianglianglee.com/2024/07/WmcixgK.png)

Name可以自选，我习惯用用IP+数据库名称

Version支持Pg13-Pg16

Connection string：数据库的链接信息，格式为

```yaml
postgresql://userName:password@hostname:5432/databaseName?sslmode=disable
```

比如

```yaml
postgresql://postgres:password@postgres:5432/pgbackweb?sslmode=disable
```

测试后保存

## 创建Destination

这里使用类S3存储都可以，比如七牛云，阿里云的OSS，自建的Minio都可以。这里以Minio为例

 ![create destination](https://static.lianglianglee.com/2024/07/ASnmxgK.png)

 Name可以自定义，我习惯用服务所在地+服务类型

Bucket Name 需要填写Minio中的Bucket

Endpoint 服务器的IP及端口，请注意这里要pgbackweb容器能访问的地址，端口使用9000，9001是管理的Web网页

Region 可随意填写

Access Key，Secret key 填写Minio的默认用户名密码即可。当然你也可以在MinioConsole再生成一个，这里不建议生成。

点击Test Connection 右下角跳出`Connection successful`代表配置没问题

## 配置Backup

 ![create backup](https://static.lianglianglee.com/2024/07/5XDnxgK.png)着重介绍几个选项

Database 选择已经创建的数据库连接

Destination 选择已创建的存储地址，多个Database可以用一个，用的Destnation directory区分即可

Cron是不带星期的格式，比如每天夜里2点 备份：`0 2 * * *`

Time zone是Cron表达式的时区，选自己所在的时区即可

Destnation directory是在S3中的文件夹，这里可以使用Database名称作为文件夹，更方便区分

Retention days 保留天数，自定义，当然也要看备份存储的成本

Options 无特殊要求默认即可


保存后，点击小闪电，即可立即执行备份

 ![backup list](https://static.lianglianglee.com/2024/07/BUROxgK.png)

## 查看备份情况


在Executions可以看到备份的执行状态。

 ![executions list](https://static.lianglianglee.com/2024/07/2bzOxgK.png)

进入Minio可以看到，文件夹名称就是在Backup中填写的文件夹

 ![minio list](https://static.lianglianglee.com/2024/07/Qz3oxgK.png)可以一层一层点进去看备份的文件，这里是tar压缩格式，可以下载后解压查看SQL文件

# 总结

pgbackweb是一个很简介，且入手难度极低的PostgreSQL备份工具，新手完全可以直接上手。基本满足了我的数据库备份需求。