---
title: Jetpack系列之Lifecycles
date: 2019-07-02
tags: [Jetpack,Lifecycles]
categories: Jetpack
toc: true
---
Lifecycle-aware components perform actions in response to a change in the lifecycle status of another component, such as activities and fragments. These components help you produce better-organized, and often lighter-weight code, that is easier to maintain.

A common pattern is to implement the actions of the dependent components in the lifecycle methods of activities and fragments. However, this pattern leads to a poor organization of the code and to the proliferation of errors. By using lifecycle-aware components, you can move the code of dependent components out of the lifecycle methods and into the components themselves.

<!--more-->

生命周期感知组件执行操作，以响应另一个组件生命周期状态的更改，例如Activity和Fragment。这些组件可以帮助您生成更有组织、更容易维护的轻量级代码。

一个常见的模式是在Activity和Fragment的生命周期方法中实现依赖组件的操作。但是，这种模式会导致代码错误的增加。通过使用生命周期感知组件，您可以将依赖组件的代码从生命周期方法转移到组件本身。


参考链接：https://developer.android.google.cn/topic/libraries/architecture/lifecycle

