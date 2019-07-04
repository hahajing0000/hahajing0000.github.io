---
title: Mvvm架构
date: 2019-07-02
tags: [架构,Mvvm]
categories: 架构
toc: true
---
{% asset_img mvvm/2019-07-04-14-19-11.png %}
<!--more-->
Mvvm架构与Mvp架构相同的是同样分为三层，并且对应层的职责功能相同：

Model层——主要负责提供数据。Model层提供数据如：网络数据及本地数据库中提取的数据，及数据结构如实体bean类。

ViewModel层——同Mvp中P层，主要负责业务逻辑的处理。通过使用官方的Databinding组件进行View层的数据更新等。ViewModel层中不包含任何View层api，使用双向绑定对View层控件进行数据更新，同样不需要View层的引用。

View层——负责界面的显示，View层只管负责UI展示不涉及任何业务逻辑，持有ViewModel层的引用。

Mvvm与Mvp的最大区别在于ViewModel层中不持有View层的引用，这样可以解耦View层，即View层的修改不会影响ViewModel层，同样使代码可测试性增强。也同样给项目团队协作提供可能，这样负责UI开发的人员和负责开发业务功能的人员可以专心关注自己的工作。

Mvvm带来的好处还有减少了很多代码，比如：findViewById 和 操作UI的代码。

**举个栗子：**


