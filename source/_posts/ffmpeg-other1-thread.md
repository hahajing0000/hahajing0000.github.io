---
title: FFmpeg系列 番外篇 Android C++ 子线程 及 生产者消费者模型
date: 2019-09-05
tags: [FFmpeg]
categories: FFmpeg
toc: true
---

这篇来讲讲Android C++ 子线程 及 生产者消费者模型

<!--more-->

Android 中的C++ 线程，我们要使用 POSIX 编写多线程 C++ 程序。

重要有三个方法：

pthread_t —— 声明线程
pthread_create —— 创建线程
pthread_exit  —— 终止线程

一个Demo来演示线程使用：
```c++
public native  void normalThread();
----------------------------------------------
pthread_t thread;

void *normalCallback(void *data){
    LOGI("create normal thread from c++");
    pthread_exit(&thread);
}

extern "C"
JNIEXPORT void JNICALL
Java_com_music_mplayer_Demo_normalThread(JNIEnv *env, jobject instance) {
    pthread_create(&thread,NULL,normalCallback,NULL);
}

```
使用起来相对简单。
