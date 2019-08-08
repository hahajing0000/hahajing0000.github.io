---
title: ARouter Demo
date: 2019-08-01
tags: [Android,组件化,ARouter]
categories: 架构
toc: true
---
这个Demo用于演示ARouter在项目中的实际使用。

本Demo建立在 [Android 组件化开发](http://www.zydeveloper.com/2019/07/30/Componentization/) 的代码基础上，大家可参考相关代码。

<!--more-->

首先，集成ARouter，在之前的代码中我们说过第三方库的集成我们一般放到Common业务组件中。
```java
 defaultConfig {
        ...
        javaCompileOptions {
            annotationProcessorOptions {
                arguments = [AROUTER_MODULE_NAME: project.getName()]
            }
        }
        ...
 }

 dependencies {
    ...
    implementation 'com.alibaba:arouter-api:1.5.0'
    annotationProcessor 'com.alibaba:arouter-compiler:1.2.2'
    ...
}
```

我们已壳工程中的MainActivity为例让其跳转到Funny业务组件中的FunnyMainActivity。

