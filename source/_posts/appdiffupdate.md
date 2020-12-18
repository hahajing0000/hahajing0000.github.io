---
title: Android APP 差分升级
date: 2020-12-18
tags: [Android,升级]
categories: Android进阶
toc: true
---

# Android App差分增量升级

## 下载并编译bsdiff

[bsdiff官网](http://www.daemonology.net/bsdiff/)

<!--more-->

如图：

![image-20200409102934958](https://i.loli.net/2020/12/18/TNGBFLhWI6UfRrV.png)

> 点击**here**



下载后的***bsdiff-4.3.tar.gz*** 即是bsdiff的源码工程了

将其上传到CentOS服务器（本地虚拟机或者Linux系统）进行编译

## 解压bsdiff-4.3.tar.gz

执行

> tar -zxvf bsdiff-4.3.tar.gz

解压后执行

> make

编译出bsdiff bspatch 可执行文件

如：

![image-20200409103750302](https://i.loli.net/2020/12/18/LrDyXINxsfv5zRQ.png)

## 制作差分升级包

bsdiff即可以用来制作**差分升级包**

如：

执行

> ./bsdiff old.apk new.apk patch

命令详解：

./bsdiff     ——  即刚编译出来的bsdiff可执行文件

old.apk    ——  源apk

new.apk  ——  新apk

patch       ——  差分升级包

执行如上命令  ***patch*** 即最终的差分升级包

## Android工程中集成bspatch

如上已经制作出了差分升级包，那么如何将差分升级包与源apk进行合并后生成新版本apk呢？

### 集成bspatch

#### 创建C++Android工程

![image-20200409104437770](https://i.loli.net/2020/12/18/YZu7p29FqnEz6jP.png)

#### 集成bspatch.c源码

将bsdiff源码中的bspatch.c文件拷贝到新建android工程的src/main/cpp目录下

![image-20200409104545334](https://i.loli.net/2020/12/18/bEvBXSWgQPMClih.png)

#### 解决bzlib错误

可能出现bzlib找不到报错的情况

![image-20200409104628873](https://i.loli.net/2020/12/18/SVEzqhXNipKvDkR.png)

这是因为bspatch.c依赖了bzlib工程，我们本地环境中不存在bzlib项目所以我们需要到其官网下载

官网下载地址：https://jaist.dl.sourceforge.net/project/bzip2/bzip2-1.0.6.tar.gz

下载后解压

在我们的android工程的src/main/cpp目录下创建bzip目录，并将刚解压的bzip源码文件拷贝到cpp/bzip下，不需要全部拷贝，只需要拷贝如下源码文件即可：

![image-20200409105427254](https://i.loli.net/2020/12/18/qhiElIKLe6npjav.png)

#### 配置CMakeList.txt

```cmake
cmake_minimum_required(VERSION 3.4.1)

message(AUTHOR_WARNING "${CMAKE_SOURCE_DIR}/bzip/")

//配置bzpi变量指定bzip下的所有c文件参与编译
file(GLOB bzip ${CMAKE_SOURCE_DIR}/bzip/*.c)

//添加bspatch 及 ${bzip}的源码到library中
add_library(
             native-lib
             SHARED
             native-lib.cpp
            bspatch.c
            ${bzip}
            )

//引入bzip下的头文件 不然会报bzlib.h找不到的错误
include_directories(${CMAKE_SOURCE_DIR}/bzip)

find_library( 
              log-lib
              log )

target_link_libraries( 
                       native-lib
                       ${log-lib} )
```

### 合并差分升级包

#### 创建BsPatcher类文件

```java
package com.zy.bsdiff;

public class BsPatcher {
    // Used to load the 'native-lib' library on application startup.
    static {
        System.loadLibrary("native-lib");
    }

    /**
     * patch 安装包
     * @param oldApk 旧版本安装包所在位置
     * @param patch  差分 patch文件
     * @param output 合成后新版本apk的输出路径
     */
    public static native void bspatch(String oldApk,String patch,String output);
}

```

如上代码中创建了native方法bspatch用于合并差分升级包，其中又三个参数：

> oldApk——旧版本安装包的所在位置
>
> patch——差分包位置
>
> ouput——合并后新版本apk输出位置

#### native-lib.cpp实现

```c++
#include <jni.h>
#include <string>
//引入patch.c中main（此次我将main改为了p_main 要同步修改一下）方法
extern "C"{
    extern int p_main(int argc,char *argv[]);
}

extern "C"
JNIEXPORT void JNICALL
Java_com_zy_bsdiff_BsPatcher_bspatch(JNIEnv *env, jclass clazz, jstring oldapk_, jstring patch_,
                                     jstring output_) {
    const char *oldApk=env->GetStringUTFChars(oldapk_,0);
    const char *patch=env->GetStringUTFChars(patch_,0);
    const char *output=env->GetStringUTFChars(output_,0);

    //创建argv参数列表  oldApk  patch  output 三个参数如上已有说明
    const char *argv[]={"",oldApk,output,patch};
    //调用patch.c中的p_main（由main方法改名为p_main）方法
    p_main(4, const_cast<char **>(argv));

    env->ReleaseStringUTFChars(oldapk_,oldApk);
    env->ReleaseStringUTFChars(patch_,patch);
    env->ReleaseStringUTFChars(output_,output);

}
```

#### 验证

MainActivity布局如下：

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity"
    android:orientation="vertical">

<TextView
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:id="@+id/txt_content"
    android:text="版本号"
    ></TextView>
<!--    <ImageView-->
<!--        android:src="@mipmap/ic_launcher"-->
<!--        android:layout_width="wrap_content"-->
<!--        android:layout_height="wrap_content">-->

<!--    </ImageView>-->
    <Button
        android:id="@+id/btn_update"
        android:text="更新"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"></Button>

</LinearLayout>
```

代码:

```java
package com.zy.bsdiff;

...

public class MainActivity extends AppCompatActivity {
    private TextView txtContent;
    private Button btnUpdate;
    private Context mContext;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mContext=this;
        initView();
        initData();
        initEvent();
        initPermission();

    }

    private void initPermission() {
        if (Build.VERSION.SDK_INT>=Build.VERSION_CODES.M){
            String perms[]={Manifest.permission.WRITE_EXTERNAL_STORAGE};
            if (checkSelfPermission(perms[0])== PackageManager.PERMISSION_DENIED){
                requestPermissions(perms,1000);
            }
        }
    }

    private void initData() {
        txtContent.setText(BuildConfig.VERSION_NAME);
    }

    private void initEvent() {
        btnUpdate.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                AsyncTask<Void, Void, File> fileAsyncTask = new AsyncTask<Void, Void, File>(){
                    @Override
                    protected void onPostExecute(File file) {
                        super.onPostExecute(file);
                        installApk(file);
                    }

                    @SuppressLint("StaticFieldLeak")
                    @Override
                    protected File doInBackground(Void... voids) {
                        //获取已安装的默认apk所在位置
                        //String oldApk=getApplicationInfo().sourceDir;
                        String oldApk = new File(Environment.getExternalStorageDirectory(),"old.apk").getAbsolutePath();
                        String patch=new File(Environment.getExternalStorageDirectory(),"patch").getAbsolutePath();
                        String output=createNewApk().getAbsolutePath();
                        //通过jin合成新apk
                        BsPatcher.bspatch(oldApk,patch,output);
                        return new File(output);
                    }
                };
                fileAsyncTask.execute();
            }


        });
    }

    /**
    *安装APK方法
    *
    /
    void installApk(File apkFile) {
        Intent intent = new Intent(Intent.ACTION_VIEW);
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {
            intent.setFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
            Uri contentUri = FileProvider.getUriForFile(mContext, "cn.bingoogolapple.update.demo.fileprovider", apkFile);
            intent.setDataAndType(contentUri, "application/vnd.android.package-archive");
        } else {
            intent.setDataAndType(Uri.fromFile(apkFile), "application/vnd.android.package-archive");
            intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        }
        if (mContext.getPackageManager().queryIntentActivities(intent, 0).size() > 0) {
            mContext.startActivity(intent);
        }
    }

    private File createNewApk() {
        File newApk=new File(Environment.getExternalStorageDirectory(),"bsdiff.apk");
        if (!newApk.exists()){
            try {
                newApk.createNewFile();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        return newApk;
    }

    private void initView() {
        txtContent = (TextView) findViewById(R.id.txt_content);
        btnUpdate = (Button) findViewById(R.id.btn_update);
    }

}

```

如上代码中很容易理解，即创建了一个AsyncTask来处理下载合并apk的任务，我们这里没有使用后台下载，直接将old.apk 及 制作的patch差分升级包 放到了内存存储跟目录中。

如下为核心代码：

```java
//获取源版本apk
String oldApk = new File(Environment.getExternalStorageDirectory(),"old.apk").getAbsolutePath();
//获取差分升级包
String patch=new File(Environment.getExternalStorageDirectory(),"patch").getAbsolutePath();
                        String output=createNewApk().getAbsolutePath();
//通过jin合成新apk
BsPatcher.bspatch(oldApk,patch,output);

//安装新的apk完成升级任务
installApk(file);
```

我们看到获取了内置存储跟目录下的old.apk 及 升级包patch  然后调用了native方法bspatch，将合并后的apk输出到“output”  然后调用了installApk来安装合并后的新版本apk。



如上及完成了简易的apk差分升级过程。



当然我们实际开发过程中old.apk不需要我们后台下方或者本地存储，可以使用如下接口api获取：

```java
String oldApk=getApplicationInfo().sourceDir;
```



