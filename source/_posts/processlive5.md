---
title: Android 进程保活（五）JobSheduler进程重生
date: 2019-07-16
tags: [Android,进程保活]
categories: Android进程保活
toc: true
---
JobSheduler在android 5.0以上版本可用，所以该方案适合5.0以上的系统版本。
关于JobService与JobSheduler不清楚的可参考官网API文档：

[官网JobService](https://developer.android.google.cn/reference/android/app/job/JobService.html)

[官网JobScheduler](https://developer.android.google.cn/reference/android/app/job/JobScheduler.html)

JobScheduler是用于计划基于应用进程的多种类型任务的api接口。当任务执行时，系统会为应用持有WakeLock，所以应用不需要做多余的确保设备唤醒的工作。

JobService继承自Service，是用于处理JobScheduler中规划的异步请求的特殊Service

<!--more-->

使用JobService必须先在AndroidManifest.xml中声明service和权限

```xml
<service android:name="MyJobService" android:permission="android.permission.BIND_JOB_SERVICE"/ >
```

我们来实现一个JobService，代码如下：
```java
package com.baweigame.mvvmdemoapplication;

import android.app.job.JobInfo;
import android.app.job.JobParameters;
import android.app.job.JobScheduler;
import android.app.job.JobService;
import android.content.ComponentName;
import android.content.Context;
import android.os.Build;
import android.support.annotation.RequiresApi;
import android.util.Log;

@RequiresApi(api = Build.VERSION_CODES.LOLLIPOP)
public class MyJobService extends JobService {
    @Override
    public void onCreate() {
        super.onCreate();
        startJob();
    }

    /**
     * 开启工作
     */
    private void startJob() {
        JobInfo.Builder builder = new JobInfo.Builder(1001, new ComponentName(getPackageName(), MyJobService.class.getName()));
        //500毫秒调用一次
        builder.setPeriodic(500);
        builder.setPersisted(true);
        JobScheduler jobScheduler = (JobScheduler) this.getSystemService(Context.JOB_SCHEDULER_SERVICE);
        jobScheduler.schedule(builder.build());
    }

    @Override
    public boolean onStartJob(JobParameters params) {
        Log.d("123", "onStartJob: ...");
        return false;
    }

    @Override
    public boolean onStopJob(JobParameters params) {
        Log.d("123", "onStopJob: ...");
        return false;
    }
}

```

在Activity启动服务，如:
```java
startService(new Intent(this,MyJobService.class));
```

我们添加了一个JobService并在服务启动及停止时加入了日志输出。
使用JobScheduler来调度服务，每500毫秒调用一次。
我们启动App来观察服务启动log的打印情况，启动后每500毫秒打印一次。我们手动关闭所有进程，发现服务停止后有复活了，正常输出了log信息。

注意：清单文件中加入相关权限。如：
```xml
 <service android:name=".MyJobService" android:permission="android.permission.BIND_JOB_SERVICE"></service>
```

---
### Android 进程保活系列：

[Android 进程保活（一）写在前面](http://www.zydeveloper.com/2019/07/15/processlive1/)
[Android 进程保活（二）双服务进程包活](http://www.zydeveloper.com/2019/07/15/processlive2/)
[Adnroid 进程保活（三）1像素方案保活](http://www.zydeveloper.com/2019/07/15/processlive3/)
[Android 进程保活（四）使用“前台服务”保活](http://www.zydeveloper.com/2019/07/16/processlive4/)
[Android 进程保活（五）JobSheduler进程重生](http://www.zydeveloper.com/2019/07/16/processlive5/)