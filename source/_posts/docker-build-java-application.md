---
title: 使用Docker部署Java程序
tags: []
categories: []
abbrlink: a1073832
date: 2023-05-17 21:59:19
---
# 背景

个人小网站升级之路中，介绍了最后使用Docker部署应用程序，本文就着重介绍怎么使用Docker部署

基本信息如下：
1. 服务器系统 Ubuntu
2. 配置 1C1G 25G SSD磁盘
3. 服务器需要安装软件：
    - Git
    - Vim


# 前期准备

**安装Docker**

https://docs.docker.com/engine/install/ubuntu/


安装完成后，记得把docker设置为自启动
```
systemctl enable docker
```


# 准备Dockerfile

因为是Java程序，需要确定自己使用了哪个版本的JDK，比如我使用的是JDK8
去dockerhub 寻找合适的基础镜像

比如我使用的是openjdk
https://hub.docker.com/_/openjdk

在tag中寻找8的版本，openjdk分为两个大版本系列
jdk 完整的编译环境
jre 运行环境，体积更小

我是用的基础镜像为`openjdk:8-jre`


```Dockerfile
# 设置本镜像需要使用的基础镜像
FROM openjdk:8-jre

# 把jar包添加到镜像中
ADD ./target/md-view-2.0.0-fat.jar /app.jar

# 添加配置文件，这个项目运行时需要
ADD config.json /config.json
# 镜像暴露的端口
EXPOSE 8081

# 修改一下文件最后修改时间
RUN bash -c 'touch /app.jar'

# 容器启动命令
ENTRYPOINT ["java","-jar","/app.jar"]

# 设置时区
RUN /bin/cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && echo 'Asia/Shanghai' >/etc/timezone
```

完整的Dockerfile如上。

# 验证Dockerfile

将项目clone下来

**Maven打包**

```
mvn clean package -DskipTests
```

**构建镜像**
注意，这里打包在项目根目录，看添加的jar文件就可以观察到需要在哪个地方开始打包


```
docker build -t md-view:1.0 .
```

**检查镜像**

```
docker run md-view:1.0
```

查看日志是否正常


# 正式启动

如上次执行的验证正常，则开始正式运行，执行后，会返回一串字符串，这个就是容器的ID


```
docker run -d -p 8081:8081 --name md-view  --restart=always -v ${pwd}:/config.json md-view:1.0
```

`--restart=always` 可以让容器异常停止时，自动启动，但是这个命令只能代表容器在运行，如容器假死，或者由于某些原因导致响应缓慢，则无法处理

如需使用真正的健康检查，需要在Dockerfile中加入健康检查指令，或者使用docker-compose


启动后，查看日志

```
docker logs -f 容器ID
```

# 其他

因为每次操作docker都需要ssh远程进机器，我一般使用 portainer 将docker操作web化，这样可以在浏览器操作docker






