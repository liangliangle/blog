---
title: 全球访问加速调研
tags: []
categories: []
abbrlink: d1ec7f99
date: 2024-06-12 22:51:13
---
# **背景**

公司的所有业务系统都部署在上海地区，北美地区访问延迟不稳定。

以下为某服务的API接口主要业务地区的初次访问

接口：`https://xxxx.com/health/check`

| **地区**   | **建连时间** | **首包时间** | **总耗时** |
| ---------- | ------------ | ------------ | ---------- |
| 加利福尼亚 | 160ms        | 165ms        | 1063ms     |
| 弗吉尼亚   | 207ms        | 220ms        | 842ms      |
| 新加坡     | 68ms         | 74ms         | 332ms      |
| 上海       | 6ms          | 19ms         | 37ms       |
| 深圳       | 34ms         | 44ms         | 303ms      |

> 数据来源：<https://boce.aliyun.com/detect/http>

除了以上耗时，还有

1. DNS解析耗时
2. SSL握手耗时
3. 下载耗时

可以看到 北美访问基本耗时在1秒以上，如果稳定的1秒钟的延迟，问题也不算特别大，然而延迟是不稳定的，比如

 ![加利福尼亚-夜间延迟数据](https://static.lianglianglee.com/2024/06/Z1WIKFK.png)

 ![弗吉尼亚-夜间延迟数据](https://static.lianglianglee.com/2024/06/cJyIKFK.png)

数据来源：阿里云监控-定时拨测 晚上9点-第二天9点的响应时间

在高峰期可以看到在晚上9点后，延迟就开始不稳定了，美西延迟波动还不算大，美东直接变为不可用了。

# 为什么会慢

说到那么慢，就需要先了解跨国流量是怎么走的[海缆地图](https://www.submarinecablemap.com/)

## 登录站

目前中国的登录站主要有以下几个地方

- [青岛](https://www.submarinecablemap.com/landing-point/qingdao-china)
- 上海
  - [崇明](https://www.submarinecablemap.com/landing-point/chongming-china)
  - [临港](https://www.submarinecablemap.com/landing-point/lingang-china)
  - [南汇](https://www.submarinecablemap.com/landing-point/nanhui-china)
- [福州](https://www.submarinecablemap.com/landing-point/fuzhou-china)
- [汕头](https://www.submarinecablemap.com/landing-point/shantou-china)
- [北海](https://www.submarinecablemap.com/landing-point/beihai-china)
- [海南](https://www.submarinecablemap.com/landing-point/hainan-china)
- [文昌](https://www.submarinecablemap.com/landing-point/wenchang-china)
- [陵水](https://www.submarinecablemap.com/landing-point/lingshui-china)

## 走向分布

中美直连的线路目前只有两条

- [TPE](https://www.submarinecablemap.com/submarine-cable/trans-pacific-express-tpe-cable-system)（跨太平洋快线）
- [NCP](https://www.submarinecablemap.com/submarine-cable/new-cross-pacific-ncp-cable-system)（新跨太平洋海缆）

往东亚方向的的

- [APCN-2](https://www.submarinecablemap.com/submarine-cable/apcn-2)（亚太2号海底光缆）
- [APG](https://www.submarinecablemap.com/submarine-cable/asia-pacific-gateway-apg)（亚太直达国际海底光缆）
- [EAC-C2C](https://www.submarinecablemap.com/submarine-cable/eac-c2c)（东亚海底光缆）
- [SJC](https://www.submarinecablemap.com/submarine-cable/southeast-asia-japan-cable-sjc)（东南亚日本海底光缆）
- [ADC](https://www.submarinecablemap.com/submarine-cable/asia-direct-cable-adc)（亚洲直线）2024年Q4投入使用
- [SJC-2](https://www.submarinecablemap.com/submarine-cable/southeast-asia-japan-cable-2-sjc2)（东南亚-日本2号海底光缆）2025年Q1投入使用

往欧洲方向

- [SEA-ME-WE-3](https://www.submarinecablemap.com/submarine-cable/seamewe-3)（亚欧3号海底光缆）
- [SEA-ME-WE-4](https://www.submarinecablemap.com/submarine-cable/seamewe-4)（亚欧4号海底光缆）
- [FEA](https://www.submarinecablemap.com/submarine-cable/flag-europe-asia-fea)（环球海底光缆）

往台湾方向

- [TSE-1](https://www.submarinecablemap.com/submarine-cable/taiwan-strait-express-1-tse-1)（海峡光缆1号）

## **为什么中国的跨国海底光缆那么少？**

因为跨国海底光缆不是一个国家的能建成的，有登陆站的国家都要参与建设，所以要**一拍即合**

拿跨太平洋快线举例，参与的企业有【AT&T，中国电信，中华电信，KT，NTT， Verizon】登陆站有6个【上海-崇明，山东-青岛，日本-东京，韩国-巨济，台湾-炭水，美国-俄勒冈】

![clipboard.png](https://static.lianglianglee.com/2024/06/iiIoKFK.png)

所以协同建造难。

晚间访问高峰时，公共线路丢包，拥堵情况严重，延迟爆炸

当然建造成本那么高，肯定要想办法盈利，故各运营方提供了付费的国际精品专线

# 加速访问

**以美国访问中国为例**

在美中之间，有两个使用高品质线路互联的服务器，两个服务器之间不存在阻塞，丢包情况。那就可以借助这两个服务器做一些改造。

美国用户访问美国的这个服务器，由这个服务器把流量打到中国的这个服务器上，再由中国的服务器向源服务器发起请求，再把响应返回出去。那就形成3段稳定的链路

![clipboard.png](https://static.lianglianglee.com/2024/06/QcNUKFK.png)

高品跨国链路，一般企业不会采购，所以购买现成的是最佳选择，阿里云就提供这样的服务

## 不能解决什么？

全球加速不能解决传输时延问题。

比如中国到美西的访问，传输时延在150毫秒左右。

## **阿里云全球加速**

 ![阿里云全球加速](https://static.lianglianglee.com/2024/06/XvfiKFK.png)

以中美间访问为例，无论是中国访问美国，还是美国访问中国，流量都是走**公共链路**。高峰期网络拥堵，丢包率、延迟都不可控且不稳定。

全球加速是中美间的流量**不走公共链路**，而是走阿里云的全球网络。基本不会产生拥堵，丢包情况，延迟可控且稳定。

## 加速方式

全球加速有三种接入方式，每种方式有不同的实现方式

1. 弹性公网IP
2. 任播弹性公网IP
3. CNAME

### 弹性公网IP

弹性公网IP，简单来说就是每个地区提供单独的公网IP，对应地区连接各自地区的IP

### 任播弹性公网IP

弹性公网IP有一个问题，每个地区都有对应的IP，没办法一个IP访问全部，如果连接到错误地区的IP，延迟更高。

这个时候借助任播技术，就可以实现一个IP指定地区的访问。

> **[任播](https://zh.wikipedia.org/wiki/%E4%BB%BB%E6%92%AD)**（英语：**anycast**）是一种网络寻址和路由的策略，使得资料可以根据路由拓扑来决定送到"最近"或"最好"的目的地。

### CNAME

CNAME实际上是套了一层的DNS解析，在阿里云的全球加速中，最后解析到的IP依旧是一个任播公网IP

# **切换流程**

DNS切换会影响全部用户，切换流程如下：

## **验证期**

- 切换CNAME解析到全球加速的域名
- TTL设置为1分钟（能选择的最短时间即可）

## **验证**

1. 清空本地DNS缓存
2. 使用dig命令解析域名
   1. dig xxxx.com
3. 本地访问`https://xxxx.com/health/check`
4. 使用加速地区的ECS或VPS等，访问 `https://xxxx.com/health/check`

## **验证完成**

- 修改TTL恢复原本时长

------

## **回滚**

CNAME解析切换回原来的即可

> 切换时间视TTL不同，有所不同，一般会略高于TTL时间