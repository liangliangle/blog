title: Debian创建Raid5阵列
tags: []
categories: []
date: 2023-06-29 01:12:23
---

# 背景
三个月前，自己组建了一个Nas服务器。使用了TrueNas的系统，使用两个U盘做RAID 1安装的系统。一直很顾虑U盘做系统的可靠性，就一直没存储重要的资料。重要资料依旧放在老Nas上。

端午过后，突然发现 boot-pool出现告警。

查看阵列情况，发现一个U盘已经挂了。

![](https://static.lianglianglee.com/assets/20230629_112301.png)

才3个月已经挂了，有点尴尬。哪怕更换了启动U盘，也不敢存放了。想着换系统。搞来搞去没有满意的系统。所以干脆裸装算了。就使用debian作为系统组软RAID,应用就万物皆docker。

# 硬盘配置

初步放了4块硬盘
480G SSD 做系统盘
3块3T HDD 组建软RAID5 

# 创建过程

**安装工具**
```bash
 apt install mdadm lvm*
```


**查找硬盘及硬盘名**
```bash
lsblk
```
输出如下：
```bash
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda      8:0    0   2.7T  0 disk
├─sda1   8:1    0     8G  0 part
└─sda2   8:2    0     2G  0 part
sdb      8:16   0   2.7T  0 disk
├─sdb1   8:17   0     8G  0 part
└─sdb2   8:18   0     2G  0 part
sdc      8:32   0 447.1G  0 disk
├─sdc1   8:33   0   512M  0 part /boot/efi
├─sdc2   8:34   0 445.7G  0 part /
└─sdc3   8:35   0   977M  0 part [SWAP]
sdd      8:48   0   2.7T  0 disk
├─sdd1   8:49   0     8G  0 part
└─sdd2   8:50   0     2G  0 part
```

需要组建的硬盘是`sda`,`sdb`,`sdd`

**组建阵列**

```bash
mdadm -Cv /dev/md0 -n 3 -l 5 -x 0 /dev/sd{a,b,d}
```

参数解释
`-C`  创建的意思
`/dev/md0`  创建的raid名字，题目要求什么就给什么
> -C下的 -a yes|no 是否自动创建目标RAID设备的设备文件

`-l` Raid的等级
`-n` 使用多少块硬盘来创建当前Raid
`-x` 空闲盘，这也就是热备盘了，当正常的盘出问题了，这块盘就能从 share状态 转换到 spare rebuilding状态，模拟情况见下方
`/dev/sd{b,c,d,e}` 这就是跟的硬盘块名字，也可以写长名字

输出
```bash
mdadm: layout defaults to left-symmetric
mdadm: layout defaults to left-symmetric
mdadm: chunk size defaults to 512K
mdadm: partition table exists on /dev/sda
mdadm: partition table exists on /dev/sda but will be lost or
       meaningless after creating array
mdadm: partition table exists on /dev/sdb
mdadm: partition table exists on /dev/sdb but will be lost or
       meaningless after creating array
mdadm: partition table exists on /dev/sdd
mdadm: partition table exists on /dev/sdd but will be lost or
       meaningless after creating array
mdadm: size set to 2930134016K
mdadm: automatically enabling write-intent bitmap on large array
Continue creating array?
Continue creating array? (y/n) y
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md0 started.
```

**查看阵列**

```bash
 mdadm -D /dev/md0
```

```bash
/dev/md0:
           Version : 1.2
     Creation Time : Thu Jun 29 01:06:55 2023
        Raid Level : raid5
        Array Size : 5860268032 (5.46 TiB 6.00 TB)
     Used Dev Size : 2930134016 (2.73 TiB 3.00 TB)
      Raid Devices : 3
     Total Devices : 3
       Persistence : Superblock is persistent

     Intent Bitmap : Internal

       Update Time : Thu Jun 29 01:07:06 2023
             State : clean, degraded, recovering
    Active Devices : 2
   Working Devices : 3
    Failed Devices : 0
     Spare Devices : 1

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : bitmap

    Rebuild Status : 0% complete

              Name : DiskStation:0  (local to host DiskStation)
              UUID : 6bf1ccaa:914b98c2:1ad546c1:6a759fc6
            Events : 3

    Number   Major   Minor   RaidDevice State
       0       8        0        0      active sync   /dev/sda
       1       8       16        1      active sync   /dev/sdb
       3       8       48        2      spare rebuilding   /dev/sdd
```

**格式化磁盘**

```bash
 mkfs.ext4 /dev/md0
```

因为磁盘原本是zfs格式，所以有提醒
```bash
mke2fs 1.47.0 (5-Feb-2023)
/dev/md0 contains a zfs_member file system
Proceed anyway? (y,N) y
Creating filesystem with 1465067008 4k blocks and 183136256 inodes
Filesystem UUID: bb3e3587-ba78-4452-98c6-08ba7671826c
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
        4096000, 7962624, 11239424, 20480000, 23887872, 71663616, 78675968,
        102400000, 214990848, 512000000, 550731776, 644972544

Allocating group tables: done
Writing inode tables: done
Creating journal (262144 blocks): done
Writing superblocks and filesystem accounting information: done
```

**挂载**
```bash
mkdir /disk
# 目录随意
mount /dev/md0 /disk
```

查看挂载结果
```bash
df -h
```

可以看到`/disk` 目录已经有大小了。

# 常用工具
因为debian不是nas专用的系统，所以需要众多的软件来替代商用nas的各种套件。这里列了一下我常用的软件。
因为是裸系统，需要自己安装，所以能用docker就用docker，安装及卸载更方便。

1. filebrowser 文件浏览器，用来替代浏览文件
2. nginx-proxy-manager 反向代理，用来整个多个端口的应用，用域名方式访问
3. minio 标准S3存储，笔记软件或者需要备份使用它
4. portainer docker管理，可以web化管理docker镜像
5. linuxserver/qbittorrent BT下载器，用作日常的资源下载使用
6. mt-photos 照片备份软件，使用方便，但是收费
7. b3log/siyuan 笔记软件，但是只能一人使用。
8. jellyfin 影音软件，用于观看视频。安卓和IOS都有可硬解的第三方客户端

开发使用的软件更多
1. MySQL
2. PostgreSQL
3. Jenkins
4. Redis
5. Jumpserver

折腾下来，这些软件距离成品nas还是有一定的距离。可是折腾的乐趣不就在此吗？

后续慢慢发现更多的开源软件吧。