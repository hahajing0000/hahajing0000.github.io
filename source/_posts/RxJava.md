---
title: RxJava使用
date: 2019-07-30
tags: [框架,RxJava]
categories: Android第三方框架
toc: true
---
### RxJava 到底是什么一个词：异步。
RxJava 在 GitHub 主页上的自我介绍是 "a library for composing asynchronous and event-based programs using observable sequences for the Java VM"（一个在 Java VM 上使用可观测的序列来组成异步的、基于事件的程序的库）。这就是 RxJava ，概括得非常精准。
<!--more-->
### RxJava 好在哪
换句话说，『同样是做异步，为什么人们用它，而不用现成的 AsyncTask / Handler / XXX / ... ？』
一个词：简洁。
异步操作很关键的一点是程序的简洁性，因为在调度过程比较复杂的情况下，异步代码经常会既难写也难被读懂。 Android 创造的 AsyncTask 和Handler ，其实都是为了让异步代码更加简洁。RxJava 的优势也是简洁，但它的简洁的与众不同之处在于，随着程序逻辑变得越来越复杂，它依然能够保持简洁。

GITHUB 地址

[Rxjava github地址](https://github.com/ReactiveX/RxJava)
[Rxandroid github地址](https://github.com/ReactiveX/RxAndroid)         

项目中依赖

```java
implementation 'io.reactivex.rxjava2:rxandroid:2.1.1'
implementation 'io.reactivex.rxjava2:rxjava:2.2.9'
```

### 一个小 demo
```java
/**
* 创建观察者
*/
Observer<String> observer = new Observer<String>() {
@Override
public void onSubscribe(Disposable d) {
    // Disposable 用于解除绑定使用 d.dispose();
    Log.d(TAG, "onSubscribe: ");
}

@Override
public void onNext(String s) {
    Log.d(TAG, "Item: " + s);
}


@Override
public void onError(Throwable e) {
    Log.d(TAG, "Error!");
}

@Override
public void onComplete() {
    Log.d(TAG, "Completed!");
}
};

// 被观察者
Observable<String> observable = Observable.create(new ObservableOnSubscribe<String>() {
@Override
public void subscribe(ObservableEmitter<String> emitter) throws Exception {
    emitter.onNext("1");
    emitter.onNext("2");
    emitter.onNext("3");
    emitter.onNext("4");
    emitter.onComplete();
}
});

// 订阅
observable.subscribe(observer);
另一个demo
DefaultSubscriber subscriber = new DefaultSubscriber<String>() {

@Override
public void onNext(String s) {
    Log.d(TAG, "Item: " + s);
}

@Override
public void onError(Throwable e) {
    Log.d(TAG, "Error!");
}

@Override
public void onComplete() {
    Log.d(TAG, "onComplete!");
}
};
Flowable<String> stringFlowable = (Flowable<String>) Flowable.create(new FlowableOnSubscribe<String>() {
@Override
public void subscribe(FlowableEmitter<String> emitter) throws Exception {
    emitter.onNext("11");
    emitter.onNext("22");
    emitter.onNext("33");
    emitter.onNext("44");
    emitter.onComplete();
}
}, BackpressureStrategy.BUFFER);
stringFlowable.subscribe(subscriber);
第3个demo
Flowable.range(0,10).subscribe(new Subscriber<Integer>() {
@Override
public void onSubscribe(Subscription s) {

}

@Override
public void onNext(Integer integer) {

}

@Override
public void onError(Throwable t) {

}

@Override
public void onComplete() {

}
});
```

### 什么是背压（Backpressure）

在RxJava中，可以通过对Observable连续调用多个Operator组成一个调用链，其中数据从上游向下游传递。当上游发送数据的速度大于下游处理数据的速度时，就需要进行Flow Control了。如果不进行Flow Control，就会抛出MissingBackpressureException异常。
这就像小学做的那道数学题：一个水池，有一个进水管和一个出水管。如果进水管水流更大，过一段时间水池就会满（溢出）。这就是没有Flow Control导致的结果。
再举个例子，在 RxJava1.x 中的 observeOn， 因为是切换了消费者的线程，因此内部实现用队列存储事件。在 Android 中默认的 buffersize 大小是16，因此当消费比生产慢时， 队列中的数目积累到超过16个，就会抛出MissingBackpressureException。

在RxJava2.0中，有五种观察者模式：
	1. Observable/Observer
	2. Flowable/Subscriber
	3. Single/SingleObserver
	4. Completable/CompletableObserver
	5. Maybe/MaybeObserver


后面三种观察者模式差不多，Maybe/MaybeObserver可以说是Single/SingleObserver和Completable/CompletableObserver的复合体。
下面列出这五个观察者模式相关的接口。

**Observable/Observer**

```java
public abstract class Observable<T> implements ObservableSource<T>{...}

public interface ObservableSource<T> {
    void subscribe(Observer<? super T> observer);
}

public interface Observer<T> {
    void onSubscribe(Disposable d);
    void onNext(T t);
    void onError(Throwable e);
    void onComplete();
}

**Completable/CompletableObserver**

//代表一个延迟计算没有任何价值,但只显示完成或异常。类似事件模式Reactive-Streams:onSubscribe(onError | onComplete)?public abstract class Completable implements CompletableSource{...}

//没有子类继承Completablepublic interface CompletableSource {
void subscribe(CompletableObserver cs);
}

public interface CompletableObserver {
    void onSubscribe(Disposable d);
    void onComplete();
    void onError(Throwable e);
}

**Flowable/Subscriber**

public abstract class Flowable<T> implements Publisher<T>{...}

public interface Publisher<T> {
    public void subscribe(Subscriber<? super T> s);
}

public interface Subscriber<T> {
    public void onSubscribe(Subscription s);
    public void onNext(T t);
    public void onError(Throwable t);
    public void onComplete();
}

**Maybe/MaybeObserver**

//Maybe类似Completable，它的主要消费类型是MaybeObserver顺序的方式，遵循这个协议:onSubscribe(onSuccess | onError | onComplete)public abstract class Maybe<T> implements MaybeSource<T>{...}

public interface MaybeSource<T> {
    void subscribe(MaybeObserver<? super T> observer);
}

public interface MaybeObserver<T> {
    void onSubscribe(Disposable d);
    void onSuccess(T t);
    void onError(Throwable e);
    void onComplete();
}

**Single/SingleObserver**

//Single功能类似于Observable,除了它只能发出一个成功的值,或者一个错误(没有“onComplete”事件)，这个特性是由SingleSource接口决定的。public abstract class Single<T> implements SingleSource<T>{...}

public interface SingleSource<T> {
    void subscribe(SingleObserver<? super T> observer);
}

public interface SingleObserver<T> {
    void onSubscribe(Disposable d);
    void onSuccess(T t);
    void onError(Throwable e);
}

```

其实从API中我们可以看到，每一种观察者都继承自各自的接口（都有一个共同的方法subscrib()），但是参数不一样），正是各自接口的不同，决定了他们功能不同，各自独立（特别是Observable和Flowable），同时保证了他们各自的拓展或者配套的操作符不会相互影响。



下面我们重点说说在实际开发中经常会用到的两个模式：
Observable/Observer和Flowable/Subscriber。

**Observable/Observer**

Observable正常用法：
```java
Observable.create(new ObservableOnSubscribe<Integer>() {
    @Override
    public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
        emitter.onNext(1);
        emitter.onNext(2);
        emitter.onComplete();
    }
    }).subscribe(new Observer<Integer>() {
    @Override
    public void onSubscribe(Disposable d) {

    }

    @Override
    public void onNext(Integer integer) {

    }

    @Override
    public void onError(Throwable e) {

    }

    @Override
    public void onComplete() {

    }
});
```

需要注意的是，这类观察模式不支持背压，下面我们具体分析下。
当被观察者快速发送大量数据时，下游不会做其他处理，即使数据大量堆积，调用链也不会报MissingBackpressureException，消耗内存过大只会OOM。
在测试的时候，快速发送了100000个整形数据，下游延迟接收，结果被观察者的数据全部发送出去了，内存确实明显增加了，遗憾的是没有OOM。
所以，当我们使用Observable/Observer的时候，我们需要考虑的是，数据量是不是很大(官方给出以1000个事件为分界线)。

**Flowable/Subscriber**
```java
Flowable.range(0, 10)
.subscribe(new Subscriber<Integer>() {
Subscription subscription;
```

```java
//当订阅后，会首先调用这个方法，其实就相当于onStart()，
//传入的Subscription s参数可以用于请求数据或者取消订阅
@Override
public void onSubscribe(Subscription s) {
    Log.d(TAG, "onsubscribe start");
    subscription = s;
    subscription.request(1);
    Log.d(TAG, "onsubscribe end");
}

@Override
public void onNext(Integer o) {
    Log.d(TAG, "onNext--->" + o);
    subscription.request(3);
}

@Override
public void onError(Throwable t) {
    t.printStackTrace();
}

@Override
public void onComplete() {
    Log.d(TAG, "onComplete");
}
});
```

输出结果如下：
onsubscribe start
onNext--->0
onNext--->1
onNext--->2
onNext--->3
onNext--->4
onNext--->5
onNext--->6
onNext--->7
onNext--->8
onNext--->9
onComplete
onsubscribe end
Flowable是支持背压的，也就是说，一般而言，上游的被观察者会响应下游观察者的数据请求，下游调用request(n)来告诉上游发送多少个数据。这样避免了大量数据堆积在调用链上，使内存一直处于较低水平。
当然，Flowable也可以通过create()来创建：
```java
Flowable.create(new FlowableOnSubscribe<Integer>() {
@Override
public void subscribe(FlowableEmitter<Integer> emitter) throws Exception {
    emitter.onNext(1);
    emitter.onNext(2);
    emitter.onNext(3);
    emitter.onComplete();
}
}, BackpressureStrategy.BUFFER);//指定背压策略
```


以下是5种背压策略：



Flowable虽然可以通过create()来创建，但是你必须指定背压的策略，以保证你创建的Flowable是支持背压的（这个在1.0的时候就很难保证，可以说RxJava2.0收紧了create()的权限）。
根据上面的代码的结果输出中可以看到，当我们调用subscription.request(n)方法的时候，不等onSubscribe()中后面的代码执行，就会立刻执行onNext方法，因此，如果你在onNext方法中使用到需要初始化的类时，应当尽量在subscription.request(n)这个方法调用之前做好初始化的工作;
当然，这也不是绝对的，我在测试的时候发现，通过create()自定义Flowable的时候，即使调用了subscription.request(n)方法，也会等onSubscribe()方法中后面的代码都执行完之后，才开始调用onNext。