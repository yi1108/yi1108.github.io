---
title: docker学习
date: 2022-11-12 12:39:44
tags:
---
# Docker安装：

https://blog.csdn.net/qq_43418737/article/details/125707321


<!--more-->


## 常用命令

### 帮助启动类命令

****

```
· 启动docker： systemctl start docker

· 停止docker： systemctl stop docker

· 重启docker： systemctl restart docker

· 查看docker状态： systemctl status docker

· 开机启动： systemctl enable docker

· 查看docker概要信息： docker info

· 查看docker总体帮助文档： docker --help

· 查看docker命令帮助文档： docker 具体命令 --help
```

### 镜像命令

· docker images

· 列出本地主机上的镜像

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/wps2.png) 

 

各个选项说明:

```
REPOSITORY：表示镜像的仓库源TAG：镜像的标签版本号IMAGE ID：镜像IDCREATED：镜像创建时间SIZE：镜像大小

 同一仓库源可以有多个 TAG版本，代表这个仓库源的不同个版本，我们使用 REPOSITORY:TAG 来定义不同的镜像。

如果你不指定一个镜像的版本标签，例如你只使用 ubuntu，docker 将默认使用 ubuntu:latest 镜像
```

· 下载镜像

```
· docker pull 镜像名字[:TAG]

· docker pull 镜像名字

· 没有TAG就是最新版

· 等价于

· docker pull 镜像名字:latest

· docker pull ubuntu
```

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/wps5.png) 

· docker system df 查看镜像/容器/数据卷所占的空间

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/wps6.png) 

· docker rmi 某个XXX镜像名字ID

· 删除镜像

#### 面试题：谈谈docker虚悬镜像是什么？

· 仓库名、标签都是<none>的镜像，俗称虚悬镜像dangling image

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/wps7.jpg)

### 容器命令



安装可视化面板 https://blog.csdn.net/weixin_46152207/article/details/125936769



设置挂载目录启动Jenkins

```
 docker run -d -v /data/jenkins/jenkins_home:/var/jenkins_home -p 9010:8080 -p 50000:50000 --restart=on-failure jenkins/jenkins:lts-jdk11

```



docker 启动nacos

```shell
docker run --name nacos -d -p 8848:8848 -p 9848:9848 -p 9849:9849 --privileged=true --restart=always -e JVM_XMS=256m -e JVM_XMX=256m -e MODE=standalone -e PREFER_HOST_MODE=hostname -e SPRING_DATASOURCE_PLATFORM=mysql -e MYSQL_SERVICE_HOST=1.15.220.36 -e MYSQL_SERVICE_PORT=3306 -e MYSQL_SERVICE_DB_NAME=nacos_config -e MYSQL_SERVICE_USER=root -e MYSQL_SERVICE_PASSWORD=123456 -v /root/apply/docker/apply/nacos/logs:/home/nacos/logs -v /root/apply/docker/apply/nacos/init.d/custom.properties:/etc/nacos/init.d/custom.properties -v /root/apply/docker/apply/nacos/data:/home/nacos/data nacos/nacos-server
```



```shell
docker 启动容器
docker run \

容器名称叫nacos -d后台运行
--name nacos -d \

nacos默认端口8848 映射到外部端口8848
-p 8848:8848 \

naocs 应该是2.0版本以后就需要一下的两个端口 所以也需要开放
-p 9848:9848 
-p 9849:9849 
--privileged=true \

docker重启时 nacos也一并重启
--restart=always \

-e 配置 启动参数
配置 jvm
-e JVM_XMS=256m 
-e JVM_XMX=256m \

单机模式
-e MODE=standalone 
-e PREFER_HOST_MODE=hostname \

数据库是mysql 配置持久化 不使用nacos自带的数据库
-e SPRING_DATASOURCE_PLATFORM=mysql \

写自己的数据库地址
-e MYSQL_SERVICE_HOST=###### \

数据库端口号
-e MYSQL_SERVICE_PORT=3306 \

mysql的数据库名称
-e MYSQL_SERVICE_DB_NAME=nacos \

mysql的账号密码
-e MYSQL_SERVICE_USER=root 
-e MYSQL_SERVICE_PASSWORD=root \

-v 映射docker内部的文件到docker外部 我这里将nacos的日志 数据 以及配置文件 映射出来
映射日志
-v /root/apply/docker/apply/nacos/logs:/home/nacos/logs \

映射配置文件 (应该没用了 因为前面已经配置参数了)
-v /root/apply/docker/apply/nacos/init.d/custom.properties:/etc/nacos/init.d/custom.properties \

映射nacos的本地数据 也没啥用因为使用了mysql
-v /root/apply/docker/apply/nacos/data:/home/nacos/data \

启动镜像名称
nacos/nacos-server
```

