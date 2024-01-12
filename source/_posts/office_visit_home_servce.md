title: 在公司无感访问家里的服务
tags: []
categories: []
date: 2024-01-11 09:05:48
---

# 在公司无感访问家里的服务

# 简述

家里有一台ALL in Boom，跑了一些服务，在家里因为可以控制DNS，可以无感访问，在公司内就没办法使用这些服务了，所以需要一种方式，在外可以访问家里的服务，并且在常用环境做到低延迟访问


想法如下：

![](https://static.lianglianglee.com/2024/01/ea2ce6a84ec56d16f05659d95a24818b.png)


**家庭环境中**：由于可以控制DNS，直接在Linux主机部署一个Nginx，然后修改路由器的域名映射，将一些自己的域名直接解析到主机上。

**其他环境中**：使用CloudFlare的tunnel，提供内网穿透及域名解析服务

**公司环境中**：因为个人经常访问，如果每次都经过cloudFlare就太慢了，所以需要用一些手段加速访问。方案是frp穿透到公网，办公电脑用nginx代理到frp端口，加速访问，这样延迟大大降低。

> 为什么将clash单独列出来，是因为host文件无法生效，需要对clash做一些改动


**域名相关**

域名使用两套

xxx.com 这个是我自购的域名
xxx.net 这个是虚构的域名，只存在局域网中

# 家庭环境

## 家庭内环境介绍：

路由器：使用刷机路由，自带frp

Linux主机：由PVE虚拟化而来，固定IP，跑各种docker服务

家庭电脑：无特殊配置，仅需连接到路由器

## 路由器配置

域名配置使用了OpenWrt的主机名配置，将需要的域名指向Linux主机

> 每个路由器配置不同

## Linux主机配置

linux主机只需要配置一个nginx代理即可。

nginx我使用的[nginx proxy manager](https://nginxproxymanager.com/)，上手简单，且可以浏览器管理。

使用docker-compose部署

```yaml
version: '3.8'
services:
  app:
    image: 'jc21/nginx-proxy-manager:2.10.4'
    restart: unless-stopped
    ports:
      - '80:80'
      - '84:81'
    volumes:
      - ./data:/data
  healthcheck:
    test: ["CMD", "/bin/check-health"]
    interval: 10s
    timeout: 3s      
```

拉起容器后，可以使用 ip:84 修改一下用户名等信息，默认用户名密码：`admin@example.com/changeme`

### 配置域名代理

代理服务的端口即可

![](https://static.lianglianglee.com/2024/01/5b02300d1df32d5f8925363e94f33105.png)

## 测试访问

使用接入路由器的任意主机，访问这些域名，看是否能访问到

# 公网环境

公网环境需要做以下准备：


1. 自购域名，并迁移到cloudflare
2. 创建一个tunnel隧道

相关文档：

* <https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/>

**tunnel国内访问速度极慢，本方法只用于紧急访问，正常情况下基本无法使用**

## 运行tunnel隧道

在页面创建完成后，会给你部署客户端的命令，这里我是用docker创建

![](https://static.lianglianglee.com/2024/01/07690de58d1c280eb668739210d28df9.png)

运行完成后，就可以看到客户端了，即完成了客户端的部署

## 配置隧道

进入隧道详情，公共主机名，添加主机名

![](https://static.lianglianglee.com/2024/01/72601465275b0e3e44f8a53d614eab20.png)

以其中的一个域名为例

subdomain：子域名，比如blog.xxx.com，blog就是子域名

domain：域名，这里选择你迁移到cloudflare的域名

service→Type：什么协议

url：从隧道客户端访问，应该用什么样的IP和端口

![](https://static.lianglianglee.com/2024/01/aaf9f5464f47d443827545b9fc1a4060.png)

创建完成后，等待几分钟，即可访问

# 办公环境

公司设置是为了加速访问，所以直接通过frp访问，

## 暴漏服务

因为有一台云上的Linux主机，所以对我来说最快的方式是使用frp将服务暴漏出来，然后通过ip:port访问服务。

> frp部署本次忽略，可自行搜索教程

frp只用暴漏家内的nginx端口即可。也就是80端口，不用一个一个将服务暴漏出来。

## Nginx配置

nginx配置的模板如下

因为这里只是代理作用，使用HTTP代理会有各种各样的配置，直接用TCP代理，能简化配置流程

```ini
stream {
    log_format proxy '$remote_addr [$time_local] '
    '$protocol $status $bytes_sent $bytes_received '
    '$session_time "$upstream_addr" '
    '"$upstream_bytes_sent" "$upstream_bytes_received" "$upstream_connect_time"';
    access_log /Nginx/logs/tcp.log proxy ;
    #将12345端口转发到192.168.1.23的3306端口
    server {
        listen 80;
        proxy_connect_timeout 5s;
        proxy_timeout 3600s;
        proxy_pass xxx.xxx.xxx.xxx:port;  # 替换成自己的ip,端口
    }
 }
```

修改`nginx/conf/nginx.conf`，这里直接使用TCP代理即可

## Host配置

windows为例，用管理员模式修改`C:\Windows\System32\drivers\etc\hosts`文件

增加以下内容

```ini
127.0.0.1 outline.lianglianglee.com
127.0.0.1 gitea.lianglianglee.com
127.0.0.1 s3.lianglianglee.com
127.0.0.1 s3-admin.lll.net
```

修改完成后，保存

> 如遇无法保存，可以把文件复制出来，改动后再复制进去

在浏览器打开配置的域名，在控制台看目标地址及端口是否是本地的nginx端口

![](https://static.lianglianglee.com/2024/01/e1d4f9bef76c8478ad6c3f4b3d7e13c6.png)

# 杂项

## clash和host不兼容的问题

办公电脑配置完成后，浏览器无法访问，但是使用curl又可以访问。

顺其自然打开了浏览器的控制台，发现请求的竟然是`127.0.0.1:7890`端口，走的clash，所以无法访问

进入clash的`Settings`/`System Proxy`/`Bypass Domin/IPNet`/`Edit`

将自定义的域名加进去即可

```ini
  - outline.lianglianglee.com
  - gitea.lianglianglee.com
  - s3.lianglianglee.com
  - s3-admin.lll.net
```

重启clash，即可访问到域名