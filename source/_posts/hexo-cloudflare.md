title: 使用Hexo+cloudflare搭建个人博客[当前博客搭建过程]
tags: []
categories: []
date: 2023-04-05 14:38:18
---
# 1 背景介绍
本人之前有一个基于tale的博客，部署在一台1C1G的云云服务器上。由于是基于Java实现，且使用了MySQL，只是博客就占用了700M的内存，基本相当于独占。后续个人基于Java+Vert.x开发了一个[技术摘抄网站](http://learn.lianglianglee.com)后，内存常年在90%以上。随着一次软件包升级，导致MySQL直接歇菜，个人博客也就随之关停了（搬到卷都后基本不更新了）。这几年随着静态博客的流行，就一直想重新搞起来一个博客。一直没有定下心，担心工作太忙无法及时更新。最近在梳理公司的各种文档及开发规范突然下定决心，这些规范可以脱敏后发布到我的博客上。而之前博客对于我来说最大的作用是强迫整理自己学到的杂乱无章的知识。



## 2 前期准备

**技术**
- Git
- Node.js
- Hexo
 
**网站**
- Github
- cloudflare

**其他**
- 域名(非必须)
- VsCode

> 如果以上的名词看不懂，读者可以先去学习这些名词及软件的用法
以上的用法本文不做赘述，百度或者bing搜一下就有大把教程


# 3 Hexo配置
Hexo的配置都在 `_config.yml` 文件中，有一个需要更改的内容

基本的配置可以参考 https://hexo.io/zh-cn/docs/configuration 很详细


## 3.1 插件配置

本人在用的几个插件
- hexo-bridge
- hexo-generator-feed
- hexo-generator-searchdb
- hexo-generator-sitemap
- hexo-wordcount


在 https://hexo.io/plugins/ 输入后半部分，即可看到插件，点进去有详细的配置说明
比如: hexo-bridge 搜索 bridge

建议新手安装 hexo-bridge ，它可以将Hexo编辑网页化，减少命令行使用


## 3.2 主题安装

本人使用的主题是 https://github.com/XPoet/hexo-theme-keep

如无需涉及主题改动，则使用npm的形式安装，如需改动则不能用npm安装

### 3.2.1 如需改动主题

需要改动主题的情况，就需要使用源码安装。主题说明给的命令有问题，导致部署时主题找不到，页面是一个白板。

需要用git子模块的形式添加进去

```bash
git submodule add https://github.com/XPoet/hexo-theme-keep.git themes/keep
```

添加完成后 修改 `_config.yml` 
```yaml
theme: keep
```
然后将  themes/keep 下的`_config.yml` 复制到项目目录，命名为 _config.keep.yml

然后按照 https://keep-docs.xpoet.cn/tutorial/configuration-guide/base_info.html 配置即可

## 3.3 本地运行

```bash
hexo server
```
然后打开 http://localhost:4000 可以看到首页。

还记得hexo-bridge吗？ 打开http://localhost:4000/bridge 即可网页编辑Hexo

注意：
 new post可能会报错，但是不影响正常的使用，报错后，刷新以下posts页面，就可以看到添加的文章了（应该是我的某个配置冲突了）


编写完成后，将代码推送到Github上

# 4 远程部署


## 4.1 构建
打开cloudflare 进去首页，右上角可将语言切换成中文
跳转到Pages

创建项目，绑定你的GitHub

绑定完成后，可以看到所有仓库选择需要部署的仓库

构建设置
- 框架预设选择 None 
- 构建命令： `hexo generate`
- 输出目录：`public`

环境变量： 
NODE_VERSION  14.3

确定后，即开始构建，构建完成后，可以在右上角看到临时域名

![](/files/assets/depoly_detail.png)


返回你的项目

![](/files/assets/cloudflare_page.png)

可以看到项目的永久域名，后续都可以使用这个域名访问

## 4.2 解析到自己的域名

记下这个域名，然后进入到你的域名管理，添加DNS记录

添加CNAME 

![](/files/assets/dns_config.png)

等待10-30分钟，即可访问


