---
title: LAME PCM转MP3
date: 2019-07-10
tags: [LAME,音视频]
categories: 音视频
toc: true
---

# LAME使用

## 集成LAME

[LAME官网](https://lame.sourceforge.io/)

[LAME下载地址（3.10.0）](https://sourceforge.net/projects/lame/files/lame/3.100/)

<!--more-->

### 开始集成

#### 源码集成

>  将下载后的源码解压
>
>  将lame-3.100\libmp3lame目录下除了.h .c的其他文件及文件夹都删除
>
>  将上面的.h .c文件拷贝到工程cpp目录下，为了便于管理我们创建一个目录来进行存放，如：
>
>  ![image-20200502094854604](https://i.loli.net/2020/12/18/hlwuCGbH6VOqABf.png)
>
>  拷贝lame源码目录下的include文件夹的lame.h到如上目录中。

#### 做一些调整

> util.h头文件中做如下修改：
>
> ![image-20200502095314612](https://i.loli.net/2020/12/18/n8hNWItKGcAdYpQ.png)
>
> 原因是android不支持红框处类型，改为float
>
> 注释掉如下代码***id3tag.c  machine.h***文件：
>
> ![image-20200502095541017](https://i.loli.net/2020/12/18/6AilxKu2E4hVBFs.png)
>
> ![image-20200502095651999](https://i.loli.net/2020/12/18/LJDE7TabVijgQIs.png)
>
> ff.c文件注释掉如下代码：
>
> ![image-20200502095811057](https://i.loli.net/2020/12/18/8DEqS9RjHelQC57.png)
>
> set_get.h 做如下修改：
>
> ![image-20200502095933467](https://i.loli.net/2020/12/18/HCLr1NjFwu82JcU.png)

#### 编写CMake文件

```cmake
cmake_minimum_required(VERSION 3.4.1)

set(SRC_DIR src/main/cpp/lamesource)
//导入头文件
include_directories(src/main/cpp/include)
#include_directories(src/main/cpp/lamesource)
//添加编译目录
aux_source_directory(src/main/cpp/lamesource SRC_LIST)

add_library(
             native-lib
             SHARED
             native-lib.cpp
             //添加编译
             ${SRC_LIST}
        )
find_library(
              log-lib
              log )
target_link_libraries(
                       native-lib
                       ${log-lib} )
```



#### build.gradle修改

```gradle
  defaultConfig {
      ...
        externalNativeBuild {
            cmake {
                //添加-D标志的意思就是给编译器添加宏定义。那么-DSTDC_HEADERS就相当于给项目增加一句"#define STDC_HEADERS"
                cppFlags "-DSTDC_HEADERS"
            }
        }
        ndk {
            abiFilters 'x86','armeabi-v7a','arm64-v8a'
        }
    }
```

#### 简单测试

```c++
#include <jni.h>
#include <string>
#include "lamesource/lame.h"

extern "C" JNIEXPORT jstring JNICALL
Java_com_android_core_ar_CppTest_stringFromJNI(
        JNIEnv* env,
        jobject /* this */) {
    std::string hello = "Hello from C++";
    //返回lame版本号
    return env->NewStringUTF(get_lame_version());
}

```



##### 可能遇到问题



machine.h

strchr 报：缺少overloadable

注释掉对应代码：

```
//char   *strchr(), *strrchr();
```

abs错误：

加入#include <stdlib.h>即可

psymodel.c            VbrTag.c            需要加入如上标注库的头文件即可