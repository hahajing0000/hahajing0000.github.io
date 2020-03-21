---
title: EventBus原理及源码分析
date: 2020-03-21
tags: [Android,源码分析]
categories: 源码分析
toc: true
---

EventBus原理及源码分析

<!--more-->

# EventBus原理及源码分析

## 一个Demo

### Demo1 普通事件的发布/接收

普通消息的发布：

```java
		/**
         * 发送普通事件
         */
        btnEventbusSend.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                //发布一个普通的事件
                EventBus.getDefault().post(new MessageEvent("我是一条普通的消息..."));
            }
        });
```

普通消息的接收：

```java
 	@Override
    protected void onStart() {
        super.onStart();
        if (!EventBus.getDefault().isRegistered(this)){
            //注册
            EventBus.getDefault().register(this);
        }

    }
	@Override
    protected void onDestroy() {
        super.onDestroy();
        if (EventBus.getDefault().isRegistered(this)){
            注销
            EventBus.getDefault().unregister(this);
        }
    } 

    //订阅方法 MessageEvent-事件类型
	@Subscribe(threadMode = ThreadMode.MAIN)
    public void onGetMessage(MessageEvent messageEvent){
        tvEventbusContent.setText(messageEvent.getContent());
    }
```

### Demo2 粘性事件发布/接收

粘性事件发布：

```java
		/**
         * 发送粘性事件
         */
        btnEventbusSendSticky.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                //发布粘性事件
                EventBus.getDefault().postSticky(new MessageEvent("我是一条粘性的消息..."));
            }
        });
```

粘性事件的接收：

与上面注册/注销代码相同，这里不再展示

```java
    @Subscribe(threadMode = ThreadMode.POSTING,sticky = true)
    public void onGetMessage(MessageEvent messageEvent){
        tvEventbusStickyContent.setText(messageEvent.getContent());
    }
```

### ThreadMode

**ThreadMode**枚举的几种方式：

- POSTING——发布者在哪个线程  订阅者就与发布者在同一个线程
- MAIN——发布者不管在哪个线程 订阅者都在主线程
- MAIN_ORDERED——与MAIN一样，不同之处是采用了队列（queue）会有顺序
- BACKGROUND——发布者如果在主线程订阅者开辟新线程接收，发布者在子线程订阅者及在发布者所在线程（等同于POSTING）
- ASYNC——发布者不管在哪个线程，订阅者都开辟新线程订阅

### EventBus的实例创建

#### getDefault()

```java
static volatile EventBus defaultInstance;

//典型的单例
public static EventBus getDefault() {
        if (defaultInstance == null) {
            synchronized (EventBus.class) {
                if (defaultInstance == null) {
                    defaultInstance = new EventBus();
                }
            }
        }
        return defaultInstance;
    }
```

使用new EventBus()来创建的实例

```java
    public EventBus() {
        this(DEFAULT_BUILDER);
    }
```

调用了this（EvnetBus）传入了DEFAULT_BUILDER，先来看一下this（DEFAULT_BUILDER）方法细节

```java
EventBus(EventBusBuilder builder) {
        logger = builder.getLogger();
        subscriptionsByEventType = new HashMap<>();
        typesBySubscriber = new HashMap<>();
        stickyEvents = new ConcurrentHashMap<>();
        mainThreadSupport = builder.getMainThreadSupport();
        mainThreadPoster = mainThreadSupport != null ? mainThreadSupport.createPoster(this) : null;
        backgroundPoster = new BackgroundPoster(this);
        asyncPoster = new AsyncPoster(this);
        indexCount = builder.subscriberInfoIndexes != null ? builder.subscriberInfoIndexes.size() : 0;
        subscriberMethodFinder = new SubscriberMethodFinder(builder.subscriberInfoIndexes,
                builder.strictMethodVerification, builder.ignoreGeneratedIndex);
        logSubscriberExceptions = builder.logSubscriberExceptions;
        logNoSubscriberMessages = builder.logNoSubscriberMessages;
        sendSubscriberExceptionEvent = builder.sendSubscriberExceptionEvent;
        sendNoSubscriberEvent = builder.sendNoSubscriberEvent;
        throwSubscriberException = builder.throwSubscriberException;
        eventInheritance = builder.eventInheritance;
        executorService = builder.executorService;
    }
```

和其他很多的框架一样，使用Builder来给EventBus相关属性进行赋值。

**DEFAULT_BUILDER**

```java
private static final EventBusBuilder DEFAULT_BUILDER = new EventBusBuilder();

EventBusBuilder() {
}
```

从上面方法我们发现直接使用的EventBusBuilder的默认设置。

#### 拓展EvnetBus的几种实例创建方式

```java
//全局只有一个当前的EventBus实例 
EventBus.getDefault();
//可以new 出来多个实例
new EventBus();
//典型的建造者模式，可以不断的持续构建
EventBus.builder().build();
```

### EventBus的register

```java
EventBus.getDefault().register(this);//这里的this就是当前activity的实例,被EventBus作为订阅者传入到EventBus中
```

如下 register方法的详细代码逻辑

```java
     /**
     * Registers the given subscriber to receive events. Subscribers must call {@link #unregister(Object)} once they
     * are no longer interested in receiving events.
     * <p/>
     * Subscribers have event handling methods that must be annotated by {@link Subscribe}.
     * The {@link Subscribe} annotation also allows configuration like {@link
     * ThreadMode} and priority.
     */
    public void register(Object subscriber) {
        //拿到入口参数的类型
        Class<?> subscriberClass = subscriber.getClass();
        //通过上面的类型subscriberClass找到其中的订阅方法进一步封装成SubscriberMethod对象
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
        synchronized (this) {
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                subscribe(subscriber, subscriberMethod);
            }
        }
    }
```

如下是根据如上Demo1中逻辑，Debug后获取到的subscriberMethods集合细节

![](C:\Users\zhangyue\Desktop\源码分析\imgs\QQ截图20200228135132.png)

我们发现在EventBusDemoActivity中我们实现了一个带@Subsribe注解的方法，然后subscriberMethods集合中出现了1个对象就是我们代码的对应方法。

#### findSubscriberMethods

SubscriberMethodFinder.java

```java
 List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
        //从缓存中获取key为subscriberClass的对应value（ List<SubscriberMethod>），METHOD_CACHE的属性定义如下
        List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
     	//如果subscriberMethods不为空直接返回该对象
        if (subscriberMethods != null) {
            return subscriberMethods;
        }

     	//如果缓存中没有对应的value就继续向下执行
        
        //我们使用getDefault单例创建的EventBus实例从而就使用的ignoreGeneratedIndex的默认值也就false 参见：EventBusBuilder -> boolean ignoreGeneratedIndex;
        if (ignoreGeneratedIndex) {
            subscriberMethods = findUsingReflection(subscriberClass);
        } else {
            //使用反射来获取subscriberMethods
            subscriberMethods = findUsingInfo(subscriberClass);
        }
        //如果subscriberMethods是empty的则直接抛异常
        if (subscriberMethods.isEmpty()) {
            throw new EventBusException("Subscriber " + subscriberClass
                    + " and its super classes have no public methods with the @Subscribe annotation");
        } else {
            //存入缓存中然后返回对应的subscriberMethods结果
            METHOD_CACHE.put(subscriberClass, subscriberMethods);
            return subscriberMethods;
        }
    }
```

如下是**METHOD_CACHE**的定义

```java
private static final Map<Class<?>, List<SubscriberMethod>> METHOD_CACHE = new ConcurrentHashMap<>();
```

##### findUsingInfo

```java
private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
        //创建或者从缓存数组中获取FindState对象，参见下方代码
        FindState findState = prepareFindState();
        //将订阅者（Demo1 - activity）赋值给findState属性
        findState.initForSubscriber(subscriberClass);
        //initForSubscriber方法核心逻辑如下代码
        //this.subscriberClass = clazz = subscriberClass;
        //clazz就一点不为空
        while (findState.clazz != null) {
            //获取定义者的信息：逻辑如果自己没有订阅者信息就查找父类 直到找到为止
            findState.subscriberInfo = getSubscriberInfo(findState);
            //第一次一定为空
            if (findState.subscriberInfo != null) {
                //找到subscriberInfo的订阅方法数组
                SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
                //迭代数组
                for (SubscriberMethod subscriberMethod : array) {
                    if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                        findState.subscriberMethods.add(subscriberMethod);
                    }
                }
            } else {
                //使用反射获取并构建SubscriberMethod的相关信息
                findUsingReflectionInSingleClass(findState);
            }
            //进一步查找分析父类
            findState.moveToSuperclass();
        }
    	//从findState实体中获取subscriberMethods 赋值给新的ArrayList对象返回 并释放findState（对象的所有属性清空）达到对象池的效果
        return getMethodsAndRelease(findState);
    }
```

###### prepareFindState

```java
//数组大小
private static final int POOL_SIZE = 4;
//FindState缓存数组
private static final FindState[] FIND_STATE_POOL = new FindState[POOL_SIZE]; 
private FindState prepareFindState() {
        synchronized (FIND_STATE_POOL) {
            //如果数组中有该对象就直接复用并清空对应数组下标位置对象
            for (int i = 0; i < POOL_SIZE; i++) {
                FindState state = FIND_STATE_POOL[i];
                if (state != null) {
                    FIND_STATE_POOL[i] = null;
                    return state;
                }
            }
        }
        //数组中没有就new创建FindSate对象
        return new FindState();
    }
```

###### findUsingReflectionInSingleClass

```java
 private void findUsingReflectionInSingleClass(FindState findState) {
        Method[] methods;
        try {
            // This is faster than getMethods, especially when subscribers are fat classes like Activities
            //获取订阅者的所有方法（当前的就是Demo1的activity）
            methods = findState.clazz.getDeclaredMethods();
        } catch (Throwable th) {
            // Workaround for java.lang.NoClassDefFoundError, see https://github.com/greenrobot/EventBus/issues/149
            methods = findState.clazz.getMethods();
            findState.skipSuperClasses = true;
        }
        //遍历得到的方法数组
        for (Method method : methods) {
            //获取到方法的访问修饰符
            int modifiers = method.getModifiers();
            //判断方法必须为public并且不能使用为如下情况
            //private static final int MODIFIERS_IGNORE = Modifier.ABSTRACT | Modifier.STATIC | BRIDGE | SYNTHETIC;
            if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
                //获取到方法的所有参数类型
                Class<?>[] parameterTypes = method.getParameterTypes();
                //如果方法参数个数是1 进入if{}逻辑
                if (parameterTypes.length == 1) {
                    //获取有Subscribe注解修饰的方法
                    Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                    if (subscribeAnnotation != null) {
                        //使用第一个参数作为事件类型使用
                        Class<?> eventType = parameterTypes[0];
                        //该方法参见下面分析 传入的是当前Method还有第一个参数类型（eventType）
                        if (findState.checkAdd(method, eventType)) {
                            //从我们自定义的订阅方法注解中获取thradMode
                            ThreadMode threadMode = subscribeAnnotation.threadMode();
                            //创建一个SubscriberMethod对象（将订阅方法的所有细节都设置到SubscriberMethod中去），赋值给findState中ArrayList（subscriberMethods）
                            findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                    subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                        }
                    }
                } 
                //如果参数个数不为1直接抛异常
                else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                    String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                    throw new EventBusException("@Subscribe method " + methodName +
                            "must have exactly 1 parameter but has " + parameterTypes.length);
                }
            } 
            //如果方法不是public或者使用了static abstract 则抛出异常
            else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                throw new EventBusException(methodName +
                        " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
            }
        }
    }
```

###### checkAdd

```java
 boolean checkAdd(Method method, Class<?> eventType) {
            // 2 level check: 1st level with event type only (fast), 2nd level with complete signature when required.
            // Usually a subscriber doesn't have methods listening to the same event type.
     		//向Map中 存储key - eventType（MessageEvent） Value - method（onGetMessage方法）
            Object existing = anyMethodByEventType.put(eventType, method);
     		//判断返回的value是否为null 当前put返回的value(existing)为空 所以直接返回了 
            if (existing == null) {
                return true;
            } else {
                //判断values是否是Method的实例 如上代码一定是
                if (existing instanceof Method) {
                    if (!checkAddWithMethodSignature((Method) existing, eventType)) {
                        // Paranoia check
                        throw new IllegalStateException();
                    }
                    // Put any non-Method object to "consume" the existing Method
                    anyMethodByEventType.put(eventType, this);
                }
                return checkAddWithMethodSignature(method, eventType);
            }
        }
```

#### subscribe

截取如上的引用方法

```java
~register 方法

synchronized (this) {
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                //调用subscribe方法并传入 subscriber -> 当前activity对象 subscriberMethod ->当前遍历的subscriberMethod
                subscribe(subscriber, subscriberMethod);
            }
        }
```

EventBus下三个数据结构如下：

```java
    //key为EventType  value为Subscription集合
    private final Map<Class<?>, CopyOnWriteArrayList<Subscription>> subscriptionsByEventType;
	//key为订阅者 value所有的事件集合
    private final Map<Object, List<Class<?>>> typesBySubscriber;
	//key为EventType的类型，value为EventType的实例
    private final Map<Class<?>, Object> stickyEvents;
```

三个数据结构创建是在EventBus的getDefault中：

<img src="eventbussource1/2020-03-21-22-26-18.png "/>


***subscribe***代码如下（对如上3个数据结构进行赋值）：

```java
    // Must be called in synchronized block
    //订阅方法
    private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
        //获取subscriberMethod的事件类型
        Class<?> eventType = subscriberMethod.eventType;
        //通过订阅者和订阅方法来构建Subscription实例
        Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
        //从subscriptionsByEventType map中获取subscriptions集合
        CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
        //获取到的集合为空 创建并添加到subscriptionsByEventType map中
        if (subscriptions == null) {
            subscriptions = new CopyOnWriteArrayList<>();
            subscriptionsByEventType.put(eventType, subscriptions);
        } else {//不为空
            //判断subscriptions中是否已经存在了newSubscription 如果存在直接抛异常
            if (subscriptions.contains(newSubscription)) {
                throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                        + eventType);
            }
        }

        //获取subscriptions大小
        int size = subscriptions.size();
        //遍历
        for (int i = 0; i <= size; i++) {
            //判断subscriberMethod优先级处理
            if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
                subscriptions.add(i, newSubscription);
                break;
            }
        }

        //从typesBySubscriber map中获取事件类型S
        List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
        //等于null 初始化ArrayList 并存入typesBySubscriber Map中
        if (subscribedEvents == null) {
            subscribedEvents = new ArrayList<>();
            typesBySubscriber.put(subscriber, subscribedEvents);
        }
        //subscribedEvents添加当前eventType
        subscribedEvents.add(eventType);

        //粘性事件的处理
        if (subscriberMethod.sticky) {
            if (eventInheritance) {
                // Existing sticky events of all subclasses of eventType have to be considered.
                // Note: Iterating over all events may be inefficient with lots of sticky events,
                // thus data structure should be changed to allow a more efficient lookup
                // (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).
                Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
                for (Map.Entry<Class<?>, Object> entry : entries) {
                    Class<?> candidateEventType = entry.getKey();
                    if (eventType.isAssignableFrom(candidateEventType)) {
                        Object stickyEvent = entry.getValue();
                        checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                    }
                }
            } else {
                //从stickyEvents map中获取stickyEvent
                Object stickyEvent = stickyEvents.get(eventType);
                //检查stickyEvent是否为空，不为空发送事件并携带Subscription
                checkPostStickyEventToSubscription(newSubscription, stickyEvent);
            }
        }
    }
```

```java
private void checkPostStickyEventToSubscription(Subscription newSubscription, Object stickyEvent) {
        if (stickyEvent != null) {
            // If the subscriber is trying to abort the event, it will fail (event is not tracked in posting state)
            // --> Strange corner case, which we don't take care of here.
            //参见Post讲解
            postToSubscription(newSubscription, stickyEvent, isMainThread());
        }
    }
```



### EventBus的unregister

```java
EventBus.getDefault().unregister(this);
```

unregister放到的代码：

```java
 /** Unregisters the given subscriber from all event classes. */
    public synchronized void unregister(Object subscriber) {
        //从typesBySubscriber map中根据传入订阅者获取对应的事件类型集合
        List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
        //如果不等于null
        if (subscribedTypes != null) {
            //迭代该集合
            for (Class<?> eventType : subscribedTypes) {
                //从subscriptionsByEventType map中移除key是eventtype的数据
                unsubscribeByEventType(subscriber, eventType);
            }
            //从typesBySubscriber中移除掉key是subscriber数据
            typesBySubscriber.remove(subscriber);
        } else {//如果为空记录warning级别log
            logger.log(Level.WARNING, "Subscriber to unregister was not registered before: " + subscriber.getClass());
        }
    }
```

unsubscribeByEventType代码如下:

```java
 private void unsubscribeByEventType(Object subscriber, Class<?> eventType) {
        List<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
        if (subscriptions != null) {
            int size = subscriptions.size();
            for (int i = 0; i < size; i++) {
                Subscription subscription = subscriptions.get(i);
                //判断subscription的订阅者等于入口传入进来的订阅者
                if (subscription.subscriber == subscriber) {
                    subscription.active = false;
                    subscriptions.remove(i);
                    i--;
                    size--;
                }
            }
        }
    }
```

#### isRegistered

该方法是用于判断是否注册

代码如下：

```java
    public synchronized boolean isRegistered(Object subscriber) {
        //从typesBySubscriber里判断是否包括key为subscriber的数据
        return typesBySubscriber.containsKey(subscriber);
    }
```

### EventBus的post

```java
    /** Posts the given event to the event bus. */
    public void post(Object event) {
        //从currentPostingThreadState （从当前线程的ThreadLocal中）获取postingState
        PostingThreadState postingState = currentPostingThreadState.get();
        //从postingState中获取eventQueue 其实eventQueue是一个ArrayList
        List<Object> eventQueue = postingState.eventQueue;
        //将外面传入的自定义事件实体加入到集合中
        eventQueue.add(event);
		//如果事件没有被发送
        if (!postingState.isPosting) {
            //是否是主线程
            postingState.isMainThread = isMainThread();
            //设置isPosting等true 默认是false
            postingState.isPosting = true;
            //判断是否已经取消如果已经取消直接抛出异常
            if (postingState.canceled) {
                throw new EventBusException("Internal error. Abort state was not reset");
            }
            try {
                //迭代事件队列
                while (!eventQueue.isEmpty()) {
                    //执行如下方法，第一个参数eventQueue.remove(0) 相当于取出索引为0的数据并删除
                    postSingleEvent(eventQueue.remove(0), postingState);
                }
            } finally {
                postingState.isPosting = false;
                postingState.isMainThread = false;
            }
        }
    }
```

#### postSingleEvent

```java
private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
    	//获取事件的类型
        Class<?> eventClass = event.getClass();
        boolean subscriptionFound = false;
        if (eventInheritance) {
            List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
            int countTypes = eventTypes.size();
            for (int h = 0; h < countTypes; h++) {
                Class<?> clazz = eventTypes.get(h);
                subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
            }
        } else {
            subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
        }
        if (!subscriptionFound) {
            if (logNoSubscriberMessages) {
                logger.log(Level.FINE, "No subscribers registered for event " + eventClass);
            }
            if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
                    eventClass != SubscriberExceptionEvent.class) {
                post(new NoSubscriberEvent(this, event));
            }
        }
    }
```

#### postSingleEventForEventType

```java
private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
        CopyOnWriteArrayList<Subscription> subscriptions;
        synchronized (this) {
            //更加当前的事件类型从subscriptionsByEventType map中获取subscriptions
            subscriptions = subscriptionsByEventType.get(eventClass);
        }
        //判断subscriptions是否为空
        if (subscriptions != null && !subscriptions.isEmpty()) { //不为空
            //迭代集合
            for (Subscription subscription : subscriptions) {
                //将传入的event实例赋值给postingState的event属性
                postingState.event = event;
                //将当前的subscription赋值给postingState.subscription
                postingState.subscription = subscription;
                //设置退出标记默认为false
                boolean aborted = false;
                try {
                    //调用如下方式传入subscription 事件实体 postingState 是否主线程的boolean值
                    postToSubscription(subscription, event, postingState.isMainThread);
                    //根据postingState.canceled;设置退出标记
                    aborted = postingState.canceled;
                } finally {
                    postingState.event = null;
                    postingState.subscription = null;
                    postingState.canceled = false;
                }
                //如果退出标记为true直接退出循环
                if (aborted) {
                    break;
                }
            }
            return true;
        }
        return false;
    }
```

#### postToSubscription

```java
private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
    //判断是哪一种线程模型
    switch (subscription.subscriberMethod.threadMode) {
        case POSTING:
            invokeSubscriber(subscription, event);
            break;
        case MAIN:
            //如果是主线程直接调用invokeSubscriber
            if (isMainThread) {
                invokeSubscriber(subscription, event);
            } else {
                //如果是子线程 使用mainThreadPoster（mainThreadPoster是HandlerPoster实例），其实enqueue操作就是使用Handler来进行线程的切换 （子线程中sendMessage 其实就是发送到 Handler handleMessage中，然后调用了eventBus.invokeSubscriber(pendingPost);）
                mainThreadPoster.enqueue(subscription, event);
            }
            break;
        case MAIN_ORDERED:
            if (mainThreadPoster != null) {
                mainThreadPoster.enqueue(subscription, event);
            } else {
                // temporary: technically not correct as poster not decoupled from subscriber
                invokeSubscriber(subscription, event);
            }
            break;
        case BACKGROUND:
            //如是主线程
            if (isMainThread) {
                //使用backgroundPoster（backgroundPoster其实实现了Runnable接口）在run方法中执行了eventBus.invokeSubscriber(pendingPost);
                //其实线程都是通过线程池来启动的eventBus.getExecutorService().execute(this);
                //eventBus中的ExecutorService默认的其实是Executors.newCachedThreadPool();当然你也可以自己定义通过EvnetBus的Builder（）
                backgroundPoster.enqueue(subscription, event);
            } else {
                //因为当前执行这段代码的线程是子线程所以invokeSubscriber就在子线程执行
                invokeSubscriber(subscription, event);
            }
            break;
        case ASYNC:
            //使用asyncPoster（asyncPoster其实实现Runnable接口），在run方法中执行eventBus.getExecutorService().execute(this);
            //其实线程都是通过线程池来启动的eventBus.getExecutorService().execute(this);
            //eventBus中的ExecutorService默认的其实是Executors.newCachedThreadPool();当然你也可以自己定义通过EvnetBus的Builder（）
            asyncPoster.enqueue(subscription, event);
            break;
        default:
            throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
    }
}
```

#### invokeSubscriber

```java
 void invokeSubscriber(Subscription subscription, Object event) {
        try {
            //此处就是利用了反射来执行我们Demo中标注为Subscribe的具体方法并传入相关参数
            subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
        } catch (InvocationTargetException e) {
            handleSubscriberException(subscription, event, e.getCause());
        } catch (IllegalAccessException e) {
            throw new IllegalStateException("Unexpected exception", e);
        }
    }
```



如上即分析完了我们使用的EventBus发送事件的详细逻辑及代码。