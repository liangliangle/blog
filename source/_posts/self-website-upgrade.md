---
title: 个人小网站的升级之路
tags: []
categories: []
abbrlink: 7d78d6bb
date: 2023-04-13 13:58:25
---
# 背景
作为一个仓鼠，从各处收集的各种文档，苦于只能PC查看，无法手机和远程查看。鉴于手头有2台云服务器，突发奇想自己写一个应用将其以web网页形式展示出来。

# 升级进程
架构升级过程分为3个阶段。基本代表了个人小网站的演变历程。
## 1.0 无保护
服务以Java jar的形式在服务器上直接运行，外层套一个nginx作端口及域名中转发
域名直接解析到云服务器，用户可以通过DNS轻而易举拿到云服务器IP，然后攻击。
当时碰到了SSH暴力破解，网站被爬虫刷流量
![](https://static.lianglianglee.com/assets/20230413_140305.png)

## CDN模式
![](https://static.lianglianglee.com/assets/20230413_140212.png)

目前常用的隐藏源站IP的方式就是在外层套一个CDN。比如阿里云的DCDN加速。其原理就是在外层再套一个nginx类服务。因为云服务厂商有更充足的带宽和计算资源，还有更充足的异常请求检查算法，比自己建设要好不少。
引入cloudflare主要原因是其有免费的基础服务，基本够我使用。引入DCDN还有一个好处就是可以查看请求量的简要信息。
接入及其简单，只需要在购买域名的平台将域名服务器切换到cloudflare提供的即可。

![](https://static.lianglianglee.com/assets/20230413_140508.png)

## 引入分析
网站部署起来后，就要想办法分析服务相关信息，比如PV，UV等等。第一反应引入第三方网站。比如google analytics。该服务国内可用。引入也很简单，只需要引入一个JS文件即可。
引入后发现其数据比较离谱，后来发现是因为浏览器自带的防跟踪功能，会屏蔽掉这些数据分析的服务商
![](https://static.lianglianglee.com/assets/20230413_140810.png)
如果真的要分析PVUV，则需要自建服务。经过一番调研，采用了umami自建服务。
使用docker-compose部署一个umami + postgreSQL。然后域名定义成 analyze.***.com，同样交给cloudflare作CDN。
使用同样简单，在网站中引入一个JS文件即可。

采集后的分析

![](https://static.lianglianglee.com/assets/20230413_140617.png)

## 自恢复
算算服务器上部署了几个程序：
1. 网站程序
2. nginx
3. umami
4. postgresql
服务器只有1C1G，Java和数据库作为吃内存大户。服务器内存已经接近满载了。如遇并发量稍微大一点，使用了较多的swap内存，整个网站就陷入超慢响应状态。需要一种资源隔离手段去解决问题。

docker这玩意正适合我，只有我的主站程序没有使用docker部署了。所以将我的网站打包成docker镜像，在服务器上运行。在docker run 命令后加入--restart always可以达到docker容器异常停止重新启动的效果。


# 后续规划

1. 开放API
2. 收集内容增加