---
title: Android插件化——动态资源加载
date: 2019-07-24
tags: [Android,插件化]
categories: Android插件化
toc: true
---
之前我们聊了一下Android插件化中的热修复，参考：（[Android 热修复](http://www.zydeveloper.com/2019/07/15/hotupdate/)），现在我们聊聊资源的动态加载。

<!--more-->

顾名思义我们之前的热更新只是解决了代码的加载，对于外部资源我们还不能直接使用。我们工程的资源最终打包后都会放到R.java文件中，然后我们可以获取Resources然后通过类似resources.getDrawable(id);方式通过id来获取我们具体的资源。但对于外部资源也就是不在我们工程中没有被放到R.java文件中的资源我们如果使用呢？