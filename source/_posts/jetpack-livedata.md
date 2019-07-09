---
title: Jetpack系列之LiveData配合ViewModel
date: 2019-07-02
tags: [Jetpack,LiveData]
categories: Jetpack
toc: true
---
**LiveData**是一个可观察的数据持有者类。与常规的可观察对象不同，LiveData是生命周期感知的，这意味着它尊重其他应用程序组件的生命周期，比如活动、片段或服务。这种意识确保LiveData只更新处于活动生命周期状态的应用程序组件观察者。

LiveData认为，如果一个观察者的生命周期处于启动或恢复状态，那么这个观察者(由observer类表示)就处于活动状态。LiveData只将更新通知活动观察者。注册为监视LiveData对象的非活动观察者不会收到有关更改的通知。您可以注册一个与实现LifecycleOwner接口的对象配对的观察者。当相应的生命周期对象的状态更改为销毁时，此关系允许删除观察者。这对于活动和片段尤其有用，因为它们可以安全地观察LiveData对象，而不用担心泄漏，当活动和片段的生命周期被破坏时，它们会立即取消订阅。

<!--more-->

