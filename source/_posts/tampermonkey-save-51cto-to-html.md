title: 油猴脚本保存51CTO为HTML文件
tags: []
categories: []
date: 2023-11-12 14:11:46
---
# 背景

因中文互联网内容经常404，所以养成了内容自己保存的习惯，看到好的内容想保存到自己本地。这次市制作51CTO的保存脚本


# 调研

51CTO分为两种内容，一种是专栏，一种是用户写的博客文章。因为博主也买一些专栏，所以也需要将其保存起来。所以本次就两种都需要使用。依旧使用油猴脚本来保存正文的HTML。

本次依旧需要再chrome或者edge中安装好油猴插件

# 开始

本次编写分为三种类型
1. 抓取专栏章节
2. 保存每个章节的内容
3. 保存用户的博客

## 专栏章节

打开一个专栏详情，可以看到详情里有章节目录，所以先把章节目录抓取过来。查看元素，发现每个章节有a标签，可以解析到章节的具体URL，span内有章节标题。所以就需要这两项内容。

先把匹配的URL规则准备好，专栏详情的页面是这种形式的 https://blog.51cto.com/cloumn/detail/123， 所以匹配规则为
```
// @match        https://blog.51cto.com/cloumn/detail/*
```

开始准备抓取URL及标题，我们用一个对象元素存储这些数据

```js
let value = {};

var itemList = document.getElementsByClassName("heading");
for (var i = 0; i < itemList.length; i++) {
    var item = itemList[i].getElementsByTagName("a")[0];
    console.log(item)
    value[item.href] = item.getElementsByTagName("span")[0].innerText
}
console.log(JSON.stringify(value))

```

可以看到抓取到了url及标题。


![image](https://static.lianglianglee.com/assets/20231130144903.png)

怎么解决存储的问题呢？因为每次刷新页面都会丢失这些数据。最简单的方案，直接用localStorage，所以需要提供一个获取和存储的方法

```js
    // 获取内容，如果没有就初始化一个空对象
    function getStorage() {
        var value = localStorage.getItem('title-list');
        if (!value) {
            value = {};
        } else {
            value = JSON.parse(value);
        }
        return value;
    }
    // 保存内容
    function saveStorage(context) {
        localStorage.setItem('title-list', JSON.stringify(value));
    }
    
    
```

抓取完成后，跳转到任意章节的详情。

```js
for (let key in value) {
    window.location.href = key;
        return;
}
```

最后尝试一下内容，是否能正确运行



## 专栏内容

专栏相对简单，因为已经有了URL和标题，我们只需要抓取正文内容即可。

专栏章节的url为https://blog.51cto.com/cloumn/blog/1223， 所以match也需要加上同样的规则。

专栏内容都在class `artical-content-bak main-content`下，我们只需要抓取第一个元素即可。


```js

  var context = document.getElementsByClassName("artical-content-bak main-content")[0].innerHTML;
```


我们依旧在开发者控制台运行一下这个脚本，能正常抓到数据。

标题可以从localStorage中获取，并且获取后，记得移除后保存，毕竟数据已经用完了。
```js
 let obj = getStorage();
 var title = obj[url];
 delete obj[url];
 saveStorage(obj);
```

最后我们保存html，我们使用一个a标签保存

```js
    function download(context, title) {
        const link = document.createElement('a');
        var textBlob = new Blob([context], { type: 'text/plain' });

        link.setAttribute('href', URL.createObjectURL(textBlob));
        link.setAttribute('download', title + ".html");
        link.click();
    }
```



## 文章

文章的URL模式为`https://blog.51cto.com/u_1234/1234`,所以先把mach规则调整好`https://blog.51cto.com/u_*/*`

其标题在H1元素下，正文就是class=container。所以整个抓取内容如下：
```js
let title = document.getElementsByTagName("h1")[0].innerText;

let context = document.getElementById("container").innerHTML;
```

同样在开发者控制台运行一次，发现没问题再整理。


# 收尾

因为涉及三个页面，我们需要判断使用哪个流程，这里就精简了，直接用if else if作为分支判断

最后脚本聚合到一起

```js
// ==UserScript==
// @name         51CTO保存
// @namespace    http://tampermonkey.net/
// @version      0.1
// @match        https://blog.51cto.com/cloumn/*
// @match   https://blog.51cto.com/u_*/*
// @description  try to take over the world!
// @author       You
// @require      https://unpkg.com/turndown/dist/turndown.js
// @require https://cdn.bootcss.com/jquery/1.12.4/jquery.min.js
// @grant        none
// ==/UserScript==

(function () {
    'use strict';

    let btn = document.createElement("button");
    btn.innerHTML = "下载";
    btn.style = "position: fixed;z-index: 999; left: 90%; top: 20px;";

    btn.onclick = function () {
        //code
        copyToMarkdown()
    }

    document.body.append(btn);
    function copyToMarkdown() {
        var url = window.location.href
        if (url.startsWith("https://blog.51cto.com/cloumn/detail")) {
            var value = getStorage();
            var itemList = document.getElementsByClassName("heading");
            for (var i = 0; i < itemList.length; i++) {
                var item = itemList[i].getElementsByTagName("a")[0];
                console.log(item)
                value[item.href] = item.getElementsByTagName("span")[0].innerText
            }
            saveStorage(value);
            for (let key in value) {
                window.location.href = key;
                return;
            }
        } else if (url.startsWith("https://blog.51cto.com/cloumn/blog")) {
            let obj = getStorage();
            var title = obj[url];
            var context = document.getElementsByClassName("artical-content-bak main-content")[0]

            delete obj[url];
            saveStorage(obj);

            download(context.innerHTML, title);
            for (let key in obj) {
                window.location.href = key;
                return;
            }

        } else if (url.startsWith("https://blog.51cto.com/u_")) {
            let title = document.getElementsByTagName("h1")[0].innerText;
           
            let context = document.getElementById("container").innerHTML;
            download(context, title);
        }
    };


    function getStorage() {
        var value = localStorage.getItem('title-list');
        if (!value) {
            value = {};
        } else {
            value = JSON.parse(value);
        }
        return value;
    }

    function saveStorage(context) {
        localStorage.setItem('title-list', JSON.stringify(context));
    }
    function download(context, title) {
        const link = document.createElement('a');
        var textBlob = new Blob([context], { type: 'text/plain' });

        link.setAttribute('href', URL.createObjectURL(textBlob));
        link.setAttribute('download', title + ".html");
        link.click();
    }
})();

```
