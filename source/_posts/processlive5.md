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

使用JobService必须先在AndroidManifest.xml中声明service和权限

```xml
<service android:name="MyJobService" android:permission="android.permission.BIND_JOB_SERVICE"/ >
```

<!--more-->

---
### Android 进程保活系列：

[Android 进程保活（一）写在前面](http://www.zydeveloper.com/2019/07/15/processlive1/)
[Android 进程保活（二）双服务进程包活](http://www.zydeveloper.com/2019/07/15/processlive2/)
[Adnroid 进程保活（三）1像素方案保活](http://www.zydeveloper.com/2019/07/15/processlive3/)
[Android 进程保活（四）使用“前台服务”保活](http://www.zydeveloper.com/2019/07/16/processlive4/)
[Android 进程保活（五）JobSheduler进程重生](http://www.zydeveloper.com/2019/07/16/processlive5/)