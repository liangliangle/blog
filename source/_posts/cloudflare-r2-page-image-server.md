title: 使用R2+Page部署免费的图床【白嫖Cloudflare】
tags: []
categories: []
date: 2024-01-15 19:17:49
---

# 背景

作为一个程序员，一般写文章都是使用markdown编写，可以快速复制到其他平台上。然而markdown有一个严重的问题，就是图片保存的问题。

刚开始我使用的typora作为编辑器，图片保存在md文件同目录的assets文件夹。一直以来都是自己阅读，后来有了在线阅读的需求，就自己编写了一个markdown阅读工具，完全根据自己的习惯编写。

后来做了博客，面临复制到博客的需求，每次编辑完，再复制图片到博客的文件夹，整个流程就繁琐了。故想一统markdown的图片存储。

现在的策略是使用picgo上传到R2上，然而这个工具有一个严重的问题，多客户端同步的问题。使用一段时间就不再使用了。

一直在想，能不能做一个网页版的图床，使用网页上传，最好不做元数据存储，直接读取R2的目录。

# 调研

程序员，就要自己**找**轮子，github上搜搜一番，找到一个还不错的工具https://github.com/roimdev/roim-picx

已经实现了这些功能，对我来说足够了

> * 图片批量上传
> * 图片列表查询
> * 图片删除
> * 目录创建
> * 按目录查询
> * 链接地址点击复制
> * 简单的身份认证功能，进入管理页面需要授权

部署上去试了试，整体对我来说很适合。其文档写的也不是很完善，就简单记录一下。

# 部署

该项目使用了Cloudflare的三个能力


1. **KV**：Token存储
2. **R2**：图片存储
3. **Page**：提供页面访问

Page中使用了functions作为后端服务

## 准备工作

**创建KV存储**

登录Cloudflare账号，`首页/Workers和Pages/KV`

创建一个命名空间

**创建R2存储**

进入`首页/R2`

创建存储桶。位置可自动

## fork仓库

略

## 部署

可按照教程部署

## 配置

因为其使用了KV和R2存储，则需要在`page详情/设置/函数`中配置这两个变量\nKV命名绑定，变量名称`XK`，KV空间选刚才创建的

R2 存储桶绑定，变量名称`PICX`，R2存储通选刚刚创建的

还需要配置一个环境变量`page详情/设置/环境变量`

新增`BASE_URL`,值空着不填即可

**Token配置**

其token是从KV存储中读取的，所以需要进入KV存储

添加条目\n密钥：`PICX_AUTH_TOKEN`，值可以自定义


**访问：**

![image.png](https://static.lianglianglee.com/2024/01/CkUQ8aK.png)

打开后，输入`PICX_AUTH_TOKEN`定义的值即可

# 魔改

 作为一个开源项目，肯定不会100%适合我，根据我的需求，做了一点点的改动。


1. 调整文件上传的存放路径为`年/月/文件`
2. 页面复制的URL为额外的域名+文件地址
3. KV只用来方Token太浪费，Token更改为环境变量存储
4. 修改网站的标题，Logo等


## 调整文件上传路径

原来的文件上传后，会传到根路径中，作为临时图床可以，如果作为静态图片存储，就不合适了，图片多了没办法处理

文件名称生成在`functions\rest\utils.ts`

修改`getFileName`为`getFilePath`，并做以下改动：

```javascript
// 获取文件名
export async function getFilePath(val: string, time: number): Promise<string> {
    const types = supportFiles.filter(it => it.type === val)
    if (!types || types.length < 1) {
        return val
    }
    const rand = Math.floor(Math.random() * 100000)
    const fileName = randomString(time + rand).concat(`.${types[0].ext}`)
    let date = new Date()
    const year = date.getFullYear() //获取完整的年份(4位)
    let month = date.getMonth() + 1 //获取当前月份(0-11,0代表1月)
    if (month < 10) {
        month = `0${month}`;
    }
    return `${year}/${month}/${fileName}`

}
```

修改调用方：

全局搜索`getFileName`，替换成`getFilePath`

部署后，测试上传是否有问题

## 增加复制地址

想法如下：

原有的图片地址不变的情况下，图片属性增加一个copyUrl的属性，在列表，上传后返回该属性，域名配置使用环境变量

**环境变更**

`functions\rest\[[path]].ts`

```javascript
export interface Env {
  BASE_URL: string
  COPY_URL: string //新增 
  XK: KVNamespace
  PICX: R2Bucket
}
```

在cloudflard的设置，增加变量

**图片对象**

```javascript
export interface ImgItem {
    key : string
    url : string
    size: number
    copyUrl: string
    filename ?: string
}
```

**API返回**

`functions\rest\routes\index.ts`

```javascript
// list接口
 return <ImgItem>{
            url: `/rest/${it.key}`,
            copyUrl: `${env.COPY_URL}/${it.key}`,
            key: it.key,
            size: it.size
  }
  // 上传接口
  urls.push({
                key: object.key,
                size: object.size,
                copyUrl: `${env.COPY_URL}/${object.key}`,
                url: `/rest/${object.key}`,
                filename: item.name
           })
  
```

**列表页面**


1. 传递参数

`src\views\ManageImages.vue`

```javascript
// image-box 增加copyUrl传递

<image-box
	:src="item.url"
    :copyUrl="item.copyUrl"
	:name="item.key"
    :size="item.size"
	@delete="deleteImage(item.key)"
	mode="uploaded"
/>
```


2. 接受入参

`src\components\ImageBox.vue`

```javascript
// 接受入参
const props = defineProps<{
	src: string
	copyUrl:string // 新增 
	name: string
	size: number
	mode: 'converted' | 'uploaded'
	uploadedAt?: number
	expiresAt?: number
}>()

---
//copyLink 方法入参修改为：copyUrl
	@click="copyLink(copyUrl)"
```

**上传页面**

`src\components\ImageItem.vue`

将`htmlLink`,`markdownLink`,`LINK`这三个地方的url改成`copyUrl`


## Token使用环境变量存储

环境变量在`functions\rest\[[path]].ts`

```javascript
export interface Env {
  AUTH_TOKEN: string
  COPY_URL: string
  R2: R2Bucket
}
```

调用方修改`functions\rest\routes\index.ts`

```javascript
await env.XK.get('PICX_AUTH_TOKEN')
// 修改为：
env.AUTH_TOKEN;
```

新增环境变量：`AUTH_TOKEN`

环境变量修改完成后，记得重新部署

## 修改网站标题

网站首页在`src\App.vue`

直接修改部署即可。


# 其他

魔改后的代码地址：<https://github.com/liangliangle/roim-picx>