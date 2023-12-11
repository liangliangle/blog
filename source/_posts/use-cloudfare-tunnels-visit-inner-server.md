title: 使用Cloudflare做HTTP内网穿透
tags: []
categories: []
date: 2023-12-11 11:39:18
---
# 背景

家里有一台24小时运行的NAS，顺便跑了一些服务。当在公司时，需要访问这些服务，就比较麻烦。传统的方法使用frp做内网穿透。该方案需要一台公网服务器，且有被攻击的风险。

方案简单对比如下：

|                | Frp                    | Cloudflare Tunnel      |
| ---------------- | ------------------------ | ------------------------ |
| 成本           | 一台公网的服务器       | 一个域名               |
| 安全           | 自己保障               | 自带简单防护           |
| 速度/延迟      | 取决与服务器带宽和延迟 | 取决内网访问国外的质量 |
| 难度           | 需要一点点linux基础    | docker即可             |
| 增加服务的方式 | 客户端修改配置文件     | 网页操作               |

**总结：**

cloudflare除了延迟之外，其他的各方面都有众多优势，特别是安全性方面优势明显。同时因为延迟的问题，不建议做远程桌面等，HTTP的服务是比较适合的。

# 前提

* 内网24小时运行的主机
* 内网可以连接到cloudflare
* 注册cloudflare
* 有一个域名，并托管到cloudflare
* 内网连接外网稳定

# 部署流程


**创建tunnels**

进入账号，点击

![](https://static.lianglianglee.com/2023/12/055870e6ac86a45a268c78bbd8154d43.png)

然后点击

![](https://static.lianglianglee.com/2023/12/c72805d17f755d660a425d9901388653.png)

点击`create tunnel`

![](https://static.lianglianglee.com/2023/12/f812578ad8bd85b376367638273380e3.png)

输入自定义的名字

![](https://static.lianglianglee.com/2023/12/57582dc975c9ba9e10260a19595a3ec9.png)

然后就将进入到了安装服务的页面

![](https://static.lianglianglee.com/2023/12/65ee22f4e3ed5c542288eeff10d3197e.png)

这里可以选择多个安装方式，因为我平时使用docker，这里就用docker作为部署方式，复制命令，然后在运行docker

> 注意：这里是前台启动的方式，需要在命令中加 -d 作为后台启动
>
> 比如
>
> ```bash
> docker run -d cloudflare/cloudflared:latest tunnel --no-autoupdate run --token  XXXXX
> ```

![](https://static.lianglianglee.com/2023/12/3898e5ae6b1640a9ef7e7a81128e7770.png)

等待服务启动后，页面下方会显示连接的客户端，点击Next

![](https://static.lianglianglee.com/2023/12/e0c0d26a8a918e81e82b5d6dc08fc13d.png)

这里就是你配置内网映射的地方了

![](https://static.lianglianglee.com/2023/12/aa7ec516b38eafb7a82d8ff3b79e2a90.png)

配置完成后，稍等1-2分钟，等待域名解析记录上传，然后就可以访问了。

# 多域名部署

tunnel并不是一个服务一个域名，可以做到一个服务映射多个服务，进入列表，点击配置

![](https://static.lianglianglee.com/2023/12/f396c7e18b20fd720f07fd2d86996543.png)

![](https://static.lianglianglee.com/2023/12/96aa554757efd0713da7f87b77d13312.png)


继续添加服务即可。