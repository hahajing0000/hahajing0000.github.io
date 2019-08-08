---
title: Android 插件化 Tinker
date: 2019-07-30
tags: [插件化,Tinker]
categories: Android插件化
toc: true
---
[Tinker 官网](http://www.tinkerpatch.com/)

### 什么是 Tinker？
Tinker 是一个开源项目([Github链接](https://github.com/Tencent/tinker))，它是微信官方的 Android 热补丁解决方案，它支持动态下发代码、So 库以及资源，让应用能够在不需要重新安装的情况下实现更新。

<!--more-->

### 为什么使用 Tinker？
当前市面的热补丁方案有很多，其中比较出名的有阿里的 AndFix、美团的 Robust 以及 QZone 的超级补丁方案。但它们都存在无法解决的问题，这也是正是推出 Tinker 的原因。
<img src="Tinker/2019-07-30-21-00-31.png">

Tinker热补丁方案不仅支持类、So 以及资源的替换，它还是2.X－7.X的全平台支持。利用Tinker我们不仅可以用做 bugfix,甚至可以替代功能的发布。**Tinker 已运行在微信的数亿 Android 设备上，那么为什么你不使用 Tinker 呢？**