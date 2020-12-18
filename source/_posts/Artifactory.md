---
title: Artifactory
date: 2019-07-30
tags: [Artifactory,数据仓库]
categories: Android进阶
toc: true
---

# 安装并使用JFrog下的Artifactory

我们使用的Artifactory是在Docker容器环境中进行安装使用

<!--more-->

## 安装Docker

[Docker下载](https://docs.docker.com/get-docker/)

下载完成后直接安装

安装后电脑桌面会出现三个图标

![image-20201112100220617](https://i.loli.net/2020/12/18/34LY6fthM7cTBRw.png)

我们双击红框位置的图标

启动可能较慢需要等一会儿。。。

![image-20201112100858976](https://i.loli.net/2020/12/18/Ky54RwuoCmjNtxa.png)

## 使用Docker获取Artifactory的影像

> ```docker
> docker pull jfrog-docker-reg2.bintray.io/jfrog/artifactory-pro:latest
> ```

## 在Docker中启动Artifactory

> docker run --name artifactory -d -p 8081:8081 -p 8082:8082 docker.bintray.io/jfrog/artifactory-oss:latest

映射两个端口 内部端口：外部端口

之后的启动可以通过 **docker start 容器id**

查看容器是否启动 ：

> docker ps

查看所有容器（包括未启动的）

> docker ps -a

![image-20201112101936082](https://i.loli.net/2020/12/18/z6FxcMCsadZ5qPf.png)

红框处就是容器id

## 访问Artifactory的web端地址

http://localhost:8081

如果localhost无法访问，我们要在docker下执行

docker-machine ip default

来查看Docker的默认ip然后使用改ip进行访问

http://192.168.99.100:8081/

## 登录Artifactory

![image-20201112103023269](https://i.loli.net/2020/12/18/t5ZgCK8w64UH9rb.png)

登录的默认用户名是admin 密码是password

登录后artifactory就会提醒你修改密码，自己修改即可

## 同步Gradle环境到Artifactory中

找到我们本地常用的gradle，我的环境位置在如下目录

D:\.gradle\wrapper\dists\gradle-6.1.1-all\cfmwm155h49vnt3hynmlrsdst

gradle-6.1.1-all.zip就是我们当前的gradle环境

将我们常用的gradle设置到artifactory上

第一步创建 仓库

![image-20201112103152901](https://i.loli.net/2020/12/18/LasTlW3peKXdbFr.png)

![image-20201112103235799](C:\Users\zhangyue\Desktop\技术库\others\images\image-20201112103235799.png)

![image-20201112103320927](https://i.loli.net/2020/12/18/Z6s2odAlRfWxjyc.png)

![image-20201112103400744](https://i.loli.net/2020/12/18/VwbWrzovjB1cC6d.png)

上传刚才的gralde环境

![image-20201112103513620](https://i.loli.net/2020/12/18/lB8TzYnPxaCpkWF.png)

![image-20201112104708717](https://i.loli.net/2020/12/18/wfbrdVmYPZg7DaA.png)

将刚才的gralde zip包拖到上面的区域就可以上传了，然后点击【Deploy】就可以了

![image-20201112104831158](https://i.loli.net/2020/12/18/lmuYeH4R2tBJEP7.png)

我们拷贝URL to file的连接地址

然后替换我们工程先的地址即可。

![image-20201112105052833](https://i.loli.net/2020/12/18/zvcwKJTaH7VIosE.png)

如上替换了Gradle的地址

## 处理第三方依赖的问题

### 创建远程仓库

![image-20201112105318116](https://i.loli.net/2020/12/18/kXMNmUH2EqZwLOD.png)

![image-20201112105345006](https://i.loli.net/2020/12/18/lWQDpq1HZbVSO3L.png)

![image-20201112105431198](https://i.loli.net/2020/12/18/f5GD79KqduXsrFH.png)

如上是演示数据不要直接使用

点击【Save&Finish】就创建了远程仓库

我们这里使用的是阿里的国内影像地址来提升下载速度

这样的仓库我们需要创建三个

![image-20201112105603517](https://i.loli.net/2020/12/18/Y13ck9rjMqERA2S.png)

https://maven.aliyun.com/repository/google

https://maven.aliyun.com/repository/jcenter 

https://maven.aliyun.com/repository/public 



### 创建虚拟仓库

![image-20201112105701137](https://i.loli.net/2020/12/18/FONVoXrbsyu1f9Z.png)

![image-20201112105719425](https://i.loli.net/2020/12/18/Vcar7BLu1l4UIA2.png)

![image-20201112105827587](https://i.loli.net/2020/12/18/9yXtG6pdeQqZH82.png)

### 使用虚拟仓库

![image-20201112110006108](https://i.loli.net/2020/12/18/CS64ptEXj2wh1cs.png)

![image-20201112110121822](https://i.loli.net/2020/12/18/z3WZHaB8pXJmOxq.png)

如上 替换原来的google jcenter仓库为我们本地的虚拟仓库

之后直接使用即可，如果有新的仓库就直接添加

## 开启允许匿名访问

![image-20201112112528295](https://i.loli.net/2020/12/18/EsPU6pV4L9uyHef.png)

