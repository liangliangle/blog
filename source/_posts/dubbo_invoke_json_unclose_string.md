title: 记一次Dubbo invoke命令的问题
tags: []
categories: []
date: 2024-03-14 11:04:12
---
# 记一次Dubbo invoke命令的问题

# 背景

因为线上发生了一次死锁问题，导致数据没有正常写入，需要手动调用dubbo invoke重试。

接口入参为 String, String, Object

命令为：

```java
invoke xxService.xxMethod("123","321",{...})
```

愉快进入容器，开始执行，不出意外，意外就发生了

```java
Invalid json argument, cause: unclosed string : 
```


# 排查

最开始以为是JSON格式有问题，开始排查格式，用了各种校验工具都没有发现问题

举例格式如下：

```json
{
    "class": "xxx",
    "xx": "xxx",
    "xxx": "xxx",
    "xxxx": "xxx",
    "xxx": "xxxx",
    "xxxRequest": {
        "xxxNo": "xxx",
        "xxxName": "xx(XXX)x"
    },
    "xxxItemDataList": [
        {
            "xxx": "xxx",
            "xxx": "xxx",
            "class": "xxx.xxx.xxx.xxx"
        }
    ]
}
```

因为接口的入参是一个父类，所以用class执行具体子类类型，

list中同样也制定了类型

用了各种工具校验，JSON都没有问题，最后跟代码发现，**是因为Json中的value有英文的括号，导致命令解析出现了问题，到括号就结束了解析**，实际到Dubbo中的JSON是

```json
{
    "class": "xxx",
    "xx": "xxx",
    "xxx": "xxx",
    "xxxx": "xxx",
    "xxx": "xxxx",
    "xxxRequest": {
        "xxxNo": "xxx",
        "xxxName": "xx(XXX
```

这可不就是JSON格式有误吗？

# 代码解析

dubbo分支切换到2.7.x

全局搜`Invalid json argument`，找到代码

 ![clipboard.png](https://static.lianglianglee.com/2024/03/Y018tCK.png)

按照第一个英文括号解析的，导致整个命令被强制截断。

再看一下最新代码 3.2的版本

 ![clipboard.png](https://static.lianglianglee.com/2024/03/3k3JtCK.png)

最新版已经修复了这个问题。


# 总结

当dubbo版本是2.7.x，使用invoke时，参数中不能有英文右括号