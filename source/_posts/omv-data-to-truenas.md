title: TrueNAS成型记
tags: []
categories: []
date: 2023-04-06 15:05:48
---
# 背景

本人19年买过一个蜗牛星际矿渣，换了电源，换成了8G内存。重装了OMV的系统，一直稳定的跑着。大部分应用都是跑docker版。孱弱的J1900 CPU也在吭哧吭哧运行着。
虽一直不满意其性能问题，但是好歹能用。最近在搞一个文档库，想要上搜索，看了看市面上应该只有ES能拿得出手，这玩意可是吃内存大户，如果搞一个可用的性能，3节点+kibana一部署，其他应用基本不可能跑了，再加上孱弱的CPU，已经远不能满足开发需求，所以另组了一台NAS，蜗牛辛苦工作了那么多年，就先歇歇吧。

**新NAS的要求**
- 6盘位起步
- 外观好看，最好能摆客厅
- CPU需要功耗小，制程越新越好
- 内存最少64G，跑各种Java应用
- 2.5G网口是必须，最好自带多网卡

**最后硬件如下**
- 机箱：QNNAS Q8 3D打印机箱
- 主板：精粤B760I
- CPU: I3 12300T
- 内存：威刚32G * 1 后续再扩展
- 数据盘：老硬盘搬过来
- 背板：咸鱼个人制作的4硬盘背板*2
- 系统：TrueNAS-SCALE
- 系统盘：32G U盘 * 2


# 过程

组装过程不表了，作为一个垃圾佬没啥难度。

## 老数据迁移

因为OMV使用的是ext4的文件系统，而TrueNAS使用的ZFS，不可能硬盘搬过来就用，所以就需要数据迁移。TrueNAS自带了 Cloud Sync Tasks，可用于数据迁移。OMV也支持SFTP，FTP等

OMV配置好SFTP，然后开始迁移，观察了一下速度：**30MB/s**，什么鬼，链路是千兆的，这连一半都没达到，然后去OMV看了一下CPU，99%。这玩意是SSH，加解密性能要求会高不少。然后去折腾FTP，发现OMV启用不了，一直报错。无果后，想尝试其他方法，突然看到了TrueNAS支持web dav。用docker再OMV搭建一个web dav的服务端，然后迁移，毕竟是基于HTTP的，少了一层的加解密性能要求会低很多。

1. 老NAS搭建web dav
```bash
docker run -d -v /srv/dev-disk-by-uuid-XXXXXX-XXX-XXX-XX/disk1:/var/webdav -e USERNAME=XXX -e PASSWORD=XXXXX -p 8888:80 morrisjobke/webdav
```

2. 新NAS配置备份凭据
3. 新增Cloud Sync Tasks (启用关闭)
4. 开始手动同步

速度：
![](https://blog-image.lianglianglee.com/assets/20230406_155750.png)

基本能跑到850Mbps以上

老NAS的CPU占用：

![](https://blog-image.lianglianglee.com/assets/old_nas_cpu_use.png)

## 应用迁移

应用迁移本着能用应用自带的迁移，就不用文件复制。最先迁移的就是思源笔记。其本身支持S3备份，所以部署一个新的docker，然后S3同步过去即可

## 新Nas部署docker

进入应用，点击`启动Docker镜像`

需要注意，在portainer中的CMD 对应TrueNAS的Container Args，而且有空格就要隔开，比如`-mode prod -workspace /siyuanworkspace -accessAuthCode password`,就要按照空格一个一个拆开

![](https://blog-image.lianglianglee.com/assets/20230406_162245.png)


因为我需要独立的IP，所以在网络使用了独立的静态IP
![](https://blog-image.lianglianglee.com/assets/20230406_161755.png)

映射一下文件夹


访问：http://{ip}:6806

**关于文件夹权限**
我习惯将Dock挂载的目录也SMB共享出来,方便修改配置之类的，TrueNAS默认不允许这样干，则需要进入`应用`-> `设置`-> `高级设置` 关闭`Enable Host Path Safety Checks`

![](https://blog-image.lianglianglee.com/assets/20230406_162603.png)

同意将docker挂载的目录分到一个docker文件夹下，大概目录是这样

```
- docker
    - siyuan 
        - data
        - config
    - mysql
        - data
        - config
    - shard
        - logs
```

方便识别和备份。如果需要共享，则会在shard中创建文件夹



## 待折腾
1. 虚拟机与主机通讯（桥接在我这不起作用，还在研究）
2. docker使用显卡硬解(12代似乎还没支持)
3. 攒钱硬盘插满




