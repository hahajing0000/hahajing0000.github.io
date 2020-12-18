---
title: 美团多渠道打包方案
date: 2019-07-10
tags: [多渠道,打包]
categories: Android进阶
toc: true
---

## 美团多渠道环境准备

### 准备Python环境

> 因为美团多渠道打包方案需要依赖到Python环境，所以第一步下载Python

[Pthon官网下载地址](https://www.python.org/downloads/)

下载后自行安装即可。

<!--more-->

### 下载美团多渠道打包工具

[Github地址](https://github.com/GavinCT/AndroidMultiChannelBuildTool)

> git下载：
>
> git clone git@github.com:GavinCT/AndroidMultiChannelBuildTool.git

![image-20201125100306961](https://i.loli.net/2020/12/18/7ClV5uwt6L4rJi9.png)

### 配置渠道

![image-20201125100417582](https://i.loli.net/2020/12/18/g1KYcWBybvkMn5o.png)

> 我们要打包的渠道在channel.txt中进行维护，每一个渠道用\r\n进行隔离，例如：

![image-20201125100505649](https://i.loli.net/2020/12/18/aFN8bdfqcPg4pwy.png)

### 打包apk进而打包多渠道apk

#### 打包我们自己的apk

![image-20201125100607355](https://i.loli.net/2020/12/18/XwCbBkSlN5quPT7.png)

等待打包结束，接下来获取到打好的包。

#### 打包多渠道包

将这个apk拷贝到PythonTool目录下，然后执行MultiChannelBuildTool.py

执行如下命令：

![image-20201125100932247](https://i.loli.net/2020/12/18/Y73dSqipeP1QyDB.png)

然后再在output-XX目录中就可以看到打好的多渠道包

![image-20201125101021463](https://i.loli.net/2020/12/18/rtkdlhgXpLe7Vw1.png)

### 获取美团打包后的渠道



![image-20201125101921917](https://i.loli.net/2020/12/18/ZMk7QnqcPCjguBf.png)

集成如上红框处的类到工程代码中

然后通过如下代码获取渠道：

```java
String channel = ChannelUtil.getChannel(MainActivity.this);
```

然后打包运行测试

到此结束！！！