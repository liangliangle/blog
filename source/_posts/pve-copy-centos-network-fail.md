title: PVE复制CentOS导致网卡启动失败的问题
tags: []
categories: []
date: 2023-11-30 14:42:03
---


# 背景

因为最近在折腾一些linux相关的软件，所以需要频繁创建Centos虚拟机，故使用了PVE的模板功能，配置好必须的软件，作为一个模板，这样虚拟机创建更快。

缺点：

- hostname一样，每个虚拟机需要改一下
- 网络因为使用的静态IP，复制完成后，需要改一下IP

修改完IP后，发现网络无法启动了

# 排查

查看日志：`journalctl -xe`​

![image](https://blog-image.lianglianglee.com/assets/image-20231129104735-a92h5s7.png)​

核心报错：

```sql
connection activation failed no suitable device found for this connection 
```

连接激活失败，未找到适合此连接的设备。

也就是说网卡找不到。

查看配置

![image](https://blog-image.lianglianglee.com/assets/image-20231129105007-obzvhzr.png)​

发现网卡物理地址有配置，而且复制虚拟机基于虚拟磁盘复制，此文件肯定不会变。故这个网卡地址是一样的。PVE复制的时候又会生成新的网卡地址。

# 修复

变更网卡地址即可，寻找PVE生成的网卡地址

![image](https://blog-image.lianglianglee.com/assets/image-20231129105152-ina6aoi.png)​

然后更新到centos配置文件中，重启网络，即可解决。