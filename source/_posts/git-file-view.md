title: 从文件看Git
tags: []
categories: []
date: 2023-10-26 11:15:09
---
# 从文件看Git

从一个提交代码说起

## 初始化仓库

为了了解Git，我们先创建一个本地仓库

```bash
git  init .
```

创建完成后，会有一个.git文件夹，打开后如下

```bash
tree .git
---------------
├─hooks
├─info
├─objects
│  ├─info
│  └─pack
└─refs
    ├─heads
    └─tags
```

一个标准的git仓库有hooks，info, objects，refs这几个文件夹，同时还有config，description，HEAD几个文件。

这里先不介绍这些是什么，我们先提交一个文件看看

这里先创建一个test.txt文件，并写入一段文本

```txt
这是第一个文件
```

使用git status查看当前仓库的状态

![image](/files/assets/image-20231023162908-77wvd7b.png)​

可以看到一个新增文件，这个时候.git目录没什么变化

## 添加文件

接下来，我们需要使用 git add 命令将文件加入到暂存区，再看git状态

![image](/files/assets/image-20231023163048-1qbqrle.png)​

使用tree命令看一下目录

```bash
├─hooks
├─info
├─objects
│  ├─e1   这个是多出来的
│  ├─info
│  └─pack
└─refs
    ├─heads
    └─tags
```

多出来一个文件夹，去看看这个文件夹是什么东西

![image](/files/assets/image-20231023163209-iri6tvb.png)​

一个无后缀的文件，打开后，是一个乱码

![image](/files/assets/image-20231023163309-3bjxq3y.png)​

这类文件可以使用 git cat-file 打开，打开这个文件，需要将两个字符的文件夹也给加上，这里就是`e1cfd5c071a3098ba096bc37300a003f0f3f036e`​

![image](/files/assets/image-20231023163527-bo6ez6n.png)​

可以看到这里是就是文本的内容

检查文件，在.git会多出来一个index文件，这个文件打不来，从字面意思可以看到是索引文件

## 提交修改

我们使用快捷的提交命令

```bash
git commit -m"第一次提交"
```

查看目录，objects中又多了两个文件夹，同时还有一个logs文件夹出来了

```bash
├─hooks
├─info
├─logs
│  └─refs
│      └─heads
├─objects
│  ├─c8
│  ├─d4
│  ├─e1
│  ├─info
│  └─pack
└─refs
    ├─heads
    └─tags
```

我们同样使用git cat-file查看objects下的文件

第一个文件，类似一个目录介绍，里面是我们add时的那个文件和文件名

![image](/files/assets/image-20231023164251-hjinoiu.png)​

另一个文件，是commit的一些信息，提交人，邮箱，时间等，其中 tree这一行，就是刚刚的文件。

![image](/files/assets/image-20231023164404-thcflji.png)​

这里就可以把整个提交流程串起来了

git add 将文件生成一个快照，保存起来

git commit 为什么会生成两个文件，还不知道为什么，我们等会再添加一个文件看看

最少我们知道其中一个文件存放着提交人之类的信息。

这时通过这一串ID，可以把整个仓库串起来

![image](/files/assets/image-20231023170406-ghlsodf.png)​

我们再加一个文件，test_bak.txt，这里同样的add和commit

```bash
├─hooks
├─info
├─logs
│  └─refs
│      └─heads
├─objects
│  ├─0b
│  ├─26
│  ├─83
│  ├─c8
│  ├─d4
│  ├─e1
│  ├─info
│  └─pack
└─refs
    ├─heads
    └─tags

```

又多了几个文件夹。使用git cat-file看看

```bash

# 第一个文件
D:\Learn\git_test>git cat-file 83911b459aa2a958469f2213ea842b5a196822c7 -p
这是第二个提交


# 第二个文件
D:\Learn\git_test>git cat-file 26e3e29b97b603085b7c0132a24a62f499666c1a -p
100644 blob e1cfd5c071a3098ba096bc37300a003f0f3f036e    test.txt
100644 blob 83911b459aa2a958469f2213ea842b5a196822c7    test_bak.txt

# 第三个文件
D:\Learn\git_test>git cat-file 0bf7f5a2d147e2ae936b248246bc77cce97bb97b -p
tree 26e3e29b97b603085b7c0132a24a62f499666c1a
parent c8cf08ad9b3d1ffd6de33dfb5976bb4535a2c488
author 李亮亮 <wb-lll538609@cainiao.com> 1698050920 +0800
committer 李亮亮 <wb-lll538609@cainiao.com> 1698050920 +0800

第二次提交

```

从这里的第二个文件看，这里甚至包含了第一个文件。在commit信息中，又多了一行parent

这个时候，关系链变成了这样：

![image](/files/assets/image-20231023170924-eeck5wr.png)​

到这里基本确定了objects的存储方式。既然是一个树，那怎么知道根节点是哪个呢？

我们可以一个一个翻文件夹

在`.git/refs/heads/master`​文件夹，存放着一个ID，这个ID就heads文件夹下对应分支的最新的commitID

还有一个问题，git怎么知道你在哪个分支呢？这个时候可以看.git目录下的HEAD文件，这里是一个引用，指向了`refs/heads/master`​。

# 其他文件夹

我们第一个提交后，多出来一个logs文件夹，我们回过头看看这个文件夹都是什么

```bash
├─logs
│  └─refs
│      └─heads
│         └─master
│  └─HEAD
```

可以看到同样有一个HEAD文件，还有一个master文件，用文本编辑器打开

```bash
0000000000000000000000000000000000000000 c8cf08ad9b3d1ffd6de33dfb5976bb4535a2c488 李亮亮 <wb-lll538609@cainiao.com> 1698050405 +0800	commit (initial): 第一次提交
c8cf08ad9b3d1ffd6de33dfb5976bb4535a2c488 0bf7f5a2d147e2ae936b248246bc77cce97bb97b 李亮亮 <wb-lll538609@cainiao.com> 1698050920 +0800	commit: 第二次提交
```

第二列，可以看到是一个ID，后边就是commit相关的内容，而这个ID就是存储commit信息的ID

第一列是上一个commit的ID。

---

index无法用文本管理器打开，怎么知道存储的是什么呢？

其实git中提供了命令可以看到index的文件

```bash
git ls-files --stage
100644 e1cfd5c071a3098ba096bc37300a003f0f3f036e 0       test.txt
100644 83911b459aa2a958469f2213ea842b5a196822c7 0       test_bak.txt
```

* 第一列 文件权限和类型
* 第二列 对象哈希
* 第三列 执行权限

  * 0 未变更或已提交
  * 1 已修改未暂存
  * 2 已暂存
  * 3 文件冲突
* 第四列 文件路径

---

接下来就是以老项目为例，一个一个解析

...

‍
