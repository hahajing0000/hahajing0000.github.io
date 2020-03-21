---
title: Handler原理及源码分析（下）
date: 2020-03-21
tags: [Android,源码分析]
categories: 源码分析
toc: true
---

Handler原理及源码分析（下）

<!--more-->



# Handler原理及源码分析（下）

## 接下来跟踪mHandler.sendMessage(message);

**sendMessage** 就是Handler中发送消息的方法。

```java
/**
     * Pushes a message onto the end of the message queue after all pending messages
     * before the current time. It will be received in {@link #handleMessage},
     * in the thread attached to this handler.
     *  
     * @return Returns true if the message was successfully placed in to the 
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.
     */ 
public final boolean sendMessage(Message msg)
    {
    	//调用如下方法 延时发送  注意一些延时毫秒数是0
        return sendMessageDelayed(msg, 0);
    }
```

```java
/**
     * Enqueue a message into the message queue after all pending messages
     * before (current time + delayMillis). You will receive it in
     * {@link #handleMessage}, in the thread attached to this handler.
     *  
     * @return Returns true if the message was successfully placed in to the 
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.  Note that a
     *         result of true does not mean the message will be processed -- if
     *         the looper is quit before the delivery time of the message
     *         occurs then the message will be dropped.
     */
    public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
        //判断如果传递进来的延时发送的long值小于0 就直接更改为0
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        //SystemClock.uptimeMillis() + delayMillis 系统时间加上延时毫秒数
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }
```

```java
    /**
     * Enqueue a message into the message queue after all pending messages
     * before the absolute time (in milliseconds) <var>uptimeMillis</var>.
     * <b>The time-base is {@link android.os.SystemClock#uptimeMillis}.</b>
     * Time spent in deep sleep will add an additional delay to execution.
     * You will receive it in {@link #handleMessage}, in the thread attached
     * to this handler.
     * 
     * @param uptimeMillis The absolute time at which the message should be
     *         delivered, using the
     *         {@link android.os.SystemClock#uptimeMillis} time-base.
     *         
     * @return Returns true if the message was successfully placed in to the 
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.  Note that a
     *         result of true does not mean the message will be processed -- if
     *         the looper is quit before the delivery time of the message
     *         occurs then the message will be dropped.
     */
    public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        //关注mQueue（消息队列）赋值
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }
```

### mQueue在Handler中如何被赋值的

我们在Demo中使用的是

```java
 private Handler mHandler=new Handler(){...}
```

来创建的主线程Handler

```java
  public Handler() {
        this(null, false);
    }
```

```java
public Handler(Callback callback, boolean async) {
        ...
        //我们发现mLooper是来源于Looper.myLooper();
        //myLooper其实就是我们ActivityThread main函数中  Looper.prepareMainLooper();
        mLooper = Looper.myLooper();
    
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread " + Thread.currentThread()
                        + " that has not called Looper.prepare()");
        }
        //mQueue 其实就是主线程的消息队列
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```

#### 下面来分析一下Looper的获取与存入

##### mLooper = Looper.myLooper();

**Looper.java**

```java
    public static @Nullable Looper myLooper() {
        //从ThreadLocalMap中获取存入的Looper对象
        return sThreadLocal.get();
    }

//sThreadLocal 是ThreadLocal其中value（T）类型是Looper
//关于ThreadLocal可以参见《ThreadLocal分析（全）》
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
```

如上代码分析了Looper.myLooper();最终获取到的是ThreadLocalMap中存入的Looper对象。

##### 那么Looper是什么时候被存入ThreadLocalMap的呢？

我们知道在子线程或者主线程中都要使用**Looper.prepare()/prepareMainLooper()**,那么prepare究竟做了什么？

```java
 public static void prepare() {
        prepare(true);
    }

    private static void prepare(boolean quitAllowed) {
        //如果ThreadLocal.get()不等于null直接抛异常
        if (sThreadLocal.get() != null) {
            //一个线程中只能有一个looper
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        //如果为空，创建一个Looper的实例作为value存入ThreadLocalMap中。
        sThreadLocal.set(new Looper(quitAllowed));
    }
```

```java
//主线程的Looper处理过程 
public static void prepareMainLooper() {
        //调用的如上方法
        prepare(false);
        synchronized (Looper.class) {
            //判断如果sMainLooper也就是主线程的Looper不为null直接抛异常
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            //将looper赋值给sMainLooper属性
            sMainLooper = myLooper();
        }
    }
```

#### 下面来分析一下mQueue = mLooper.mQueue;

如上代码比较简单，直接是将Looper中的mQueue属性赋值给了Handler的mQueue属性。

下面我们来分析一下Looper中的mQueue是什么时候被创建的？

如上我们分析Looper.prepare()方法时，我们发现了创建Looper的代码，如：

```java
//如果为空，创建一个Looper的实例作为value存入ThreadLocalMap中。
sThreadLocal.set(new Looper(quitAllowed));
```

如上代码创建了Looper并执行了Looper（boolean quitAllowed）的构造方法，下面我们来看一下这个构造：

```java
private Looper(boolean quitAllowed) {
    	//创建并初始化了消息队列
        mQueue = new MessageQueue(quitAllowed);
    	//设置了Looper的mThread属性为当前线程
        mThread = Thread.currentThread();
    }
```

我们发现消息队列是在***Looper***中初始化的，下面来看看MessageQueue

```java
    private native static long nativeInit();
    private native static void nativeDestroy(long ptr);
    //处理消息队列中的消息，没有消息则阻塞
    private native void nativePollOnce(long ptr, int timeoutMillis); /*non-static for callbacks*/
    //唤醒
    private native static void nativeWake(long ptr);
    private native static boolean nativeIsPolling(long ptr);
    private native static void nativeSetFileDescriptorEvents(long ptr, int fd, int events);

    //构造函数比较简单
    MessageQueue(boolean quitAllowed) {
        mQuitAllowed = quitAllowed;
        mPtr = nativeInit();
    }
```

上面是MessageQueue中的一些本地方法及构造函数

如下是MessageQueue的两个重要方法：

//向消息队列（实际是**单向链表**）中存入消息，调用位置：***sendMessageAtTime***

```java
boolean enqueueMessage(Message msg, long when){
     if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        synchronized (this) {
                ......
                //存入数据
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
}
```

//取出并删除消息，调用位置：***Looper.loop()*** 

```java
Message next() {
        ......
        //死循环
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }

            //有消息则取出 没有则堵塞
            nativePollOnce(ptr, nextPollTimeoutMillis);

            ......
        }
}
```

我们想next方法其实开启了死循环一直执行，其中的nativePollOnce方法是本地方法用于获取消息队列中的消息，没有消息则堵塞，这也是为什么Handler机制不耗性能及不会卡顿的关键（底层使用了pipe命名管道 及epoll I/O模型详细可以参见：[Android 消息处理以及epoll机制](https://www.jianshu.com/p/97e6e6c981b6)）

#### 下面来分析一下Looper.loop()

```java
/**
     * Run the message queue in this thread. Be sure to call
     * {@link #quit()} to end the loop.
     */
    public static void loop() {
        //获取线程上的Looper对象
        final Looper me = myLooper();
        //如果Looper对象me 为空则抛出异常信息，原因：没有调用Looper.prepare()
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        //从Looper上获取消息队列赋值给局部变量queue
        final MessageQueue queue = me.mQueue;

        ...

        //开启死循环提取消息
        for (;;) {
            //使用MessageQueue对象的next方法读取消息，详见如上对next的源码分析
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            ...
                
 			//分发的开始时间戳
            final long dispatchStart = needStartTime ? SystemClock.uptimeMillis() : 0;
            //分发的结束时间戳
            final long dispatchEnd;
            try {
                //分发消息
                msg.target.dispatchMessage(msg);
                dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }
            
            ...

            msg.recycleUnchecked();
        }
    }
```

##### 分析一下msg.target.dispatchMessage(msg);

```java
/**
     * Handle system messages here.
     */
    public void dispatchMessage(Message msg) {
        //如果msg对象的callback不为空，直接使用handleCallback(msg)
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            //如果handler的mCallback不为空的情况
            if (mCallback != null) {
                //将消息msg交给mCallback处理，如果返回true直接return，如果返回了false继续向下执行也就是会执行到如下的Handler自己的handleMessage（msg）；方法
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            //Handler本身对分发过来的msg处理
            handleMessage(msg);
        }
    }
```

```java
//如果msg对象的callback不为空的情况，会执行如下方法，很简单：message的callback是Runnable，所以执行run方法就ok了 
private static void handleCallback(Message message) {
        message.callback.run();
    }
```

mCallback不为空的Demo代码如下：

```java
private Handler mHandler=new Handler(new Handler.Callback() {
        @Override
        public boolean handleMessage(Message msg) {
            ...
            return false;
        }
    })
    {
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            ...
        }
    };
```

如下是Handler接口Callback的定义：

```java
    public interface Callback {
        /**
         * @param msg A {@link android.os.Message Message} object
         * @return True if no further handling is desired
         */
        public boolean handleMessage(Message msg);
    }
```

如上handleMesssage就是由我们开发人员自己实现的消息处理逻辑了。

上面所述就是Handler机制的核心原理及核心源码部分。