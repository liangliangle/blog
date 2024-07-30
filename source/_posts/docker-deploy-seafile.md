---
title: Docker部署Seafile服务
tags: []
categories: []
abbrlink: 1f9eb31a
date: 2023-07-04 19:58:00
---
# 背景
作为一台半AIO的NAS，除了使用SMB挂在外，还需要有一个云盘管理软件。
常见的有
- owncloud
- nextcloud

以上两个服务太重了，不太适合我。需要一个轻量级的服务。尽可能少的占用资源。最后选择了seafile

其可以做到文件夹无感同步。这点对于我来说比较重要。

# 先决条件
因为采用docker部署，所以需要安装docker， docker-compose。

如果采用自己的mysql服务，则需要准备一个mysql服务。并需要root账户密码。


# 部署脚本
```yaml
version: '2.0'
services:

  memcached:
    image: memcached:1.6
    container_name: seafile-memcached
    entrypoint: memcached -m 256
          
  seafile:
    image: seafileltd/seafile-mc:latest
    container_name: seafile
    ports:
      - "85:80"
    volumes:
      - /disk/docker/seafile/data:/shared 
    environment:
      - DB_HOST=********
      - DB_ROOT_PASSWD=****
      - TIME_ZONE=Asia/Shanghai 
      - SEAFILE_ADMIN_EMAIL=***@sina.com
      - SEAFILE_ADMIN_PASSWORD=***
      - SEAFILE_SERVER_LETSENCRYPT=false   
      - SEAFILE_SERVER_HOSTNAME=***:85
    depends_on:
      - memcached

```

等待拉取镜像，启动。

因为一些原因，会导致**admin账号创建失败**

```log
Seahub is started
Error happened during creating seafile admin.
Starting seahub at port 8000 ...
```

碰到这个报错不要慌。因为seafile自动生成的数据库密码有问题，导致的
1. 查询seafile生成的密码

```bash
docker  exec -it seafile cat /shared/seafile/conf/seafile.conf | grep password
```

```bash
password = 5b1fd1f4-f8f2-45a8-aa4e-*****
```

这里显示的就是seafile生成的密码

2. 把数据库密码替换掉

注意把密码换成查到的
```sql
ALTER USER 'seafile'@'%.%.%.%' IDENTIFIED BY '5b1fd1f4-f8f2-45a8-aa4e-***' PASSWORD EXPIRE NEVER;
ALTER USER 'seafile'@'%.%.%.%' IDENTIFIED WITH mysql_native_password BY '5b1fd1f4-f8f2-45a8-aa4e-****';
FLUSH PRIVILEGES;
ALTER USER 'seafile'@'%.%.%.%' IDENTIFIED BY '5b1fd1f4-f8f2-45a8-aa4e-***';
```

3. 创建管理员账号

```bash
docker exec -it seafile /opt/seafile/seafile-server-latest/reset-admin.sh
```

按照提示，输入邮箱，密码(需要两遍)

然后打开 http://ip:85

使用创建的管理员账号登陆即可。
