---
title: Java SPI
date: 2020-12-18
tags: [Android,SPI]
categories: Java进阶
toc: true
---

# Java SPI

## 什么是SPI

英文是Service Provider Interface，是Java 内置的服务发现机制。

SPI在日常Android开发中比较少使用，但在模块化开发中会比较适用。

一般在开发过程中，我们可以将业务抽取成API 接口，可以为API提供不同的业务实现 。这种实现可以在不同的模块或者jar aar中。可以通过不同的实现来动态扩展业务功能甚至实现插件一样的可插拔使用。

SPI的最终目的就是解耦！！！

<!--more-->

## 如何使用SPI

> SPI使用方法并不复杂，一般3步就可以实现
>
> 1、定义一个API接口
>
> 2、提供方（实现方）创建java同级目录创建resources目录，创建META-INF/services子目录，目录下放置如上接口的完整限定名的文件，文件中放置具体实现类完整限定名，多个用回车换行分隔
>
> 3、使用方（调用方）通过ServiceLoader.load方法加载具体实现类实例即可

## 一个使用SPI的例子

### 小需求

***编写一个用于动态装配模块进行具体日志输出的小Demo***

### 项目结构

先看看工程目录：

![image-20201201133712888](https://i.loli.net/2020/12/18/FiTSrCMpIk3VLcW.png)



- 1   ——   使用方
- 2   ——   实现方 1
- 3   ——   实现方 2
- 4   ——   API接口



如上工程目录中，我们发现共创建4个Moudle

app是使用方即具体使用业务的模块，依赖Interface模块也依赖Do1 Do2模块

Interface是定义API业务接口的模块

Do1 Do2 为实现了Interface的具体实现方，其中是不同的业务实现，即Do1 Do2 依赖了Interface模块



### 具体代码实现

#### Interface模块

> Interface模块创建了Do接口

```java
/**
 * @author:zhangyue
 * @date:2020/12/1
 */
public interface Do {
    void doSomeing();
}

```

创建了一个方法，dosomeing做一些事情

#### Do1模块

首先Do1依赖Interface模块

实现Do接口

```java
/**
 * @author:zhangyue
 * @date:2020/12/1
 */
public class Do1Impl implements Do {
    @Override
    public void doSomeing() {
        Log.d("123","Do1 Impl ....");
    }
}
```

创建如下目录结构

![image-20201201134636524](https://i.loli.net/2020/12/18/8pBJzg2HyILfG3s.png)

即Java同级创建 resources\META-INF\services

在services目录下创建一个文件，名字为上面Interface模块中创建的接口完全限定名 这里为：“com.zy.aninterface.Do”



文件中将该模块的接口实现类完整限定名加入，如：

```txt
com.zy.do1.Do1Impl
```

这样就完成了该模块的代码及相关SPI设置

#### Do2模块

与Do1模块一样，同步修改为Do2的实现类名即可。

#### app模块

App模块即使用方，Activity中添加一个Button用于演示

```java
public class MainActivity extends AppCompatActivity {
    private Button btnTest;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);


        btnTest = (Button) findViewById(R.id.btn_test);


        btnTest.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                ServiceLoader<Do> loader=ServiceLoader.load(Do.class);
                Iterator<Do> iterator = loader.iterator();
                while (iterator.hasNext()){
                    iterator.next().doSomeing();
                }
            }
        });
    }

}
```

**着重看**ServiceLoader.load

该方法即可加载每个Do接口的具体实现类

代码中使用迭代器进行迭代并调用具体业务实现类的dosomeing方法。

当然，具体的业务实现类app模块需要依赖导入才可以。



如上就是SPI的简单使用，具体的还要根据各自业务具体实现。



