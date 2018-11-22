---
title: EventBus源码解读
date: 2016年01月12日 10:25:29
comments: true
toc: true
categories: 移动开发
tags: [EventBus,Android,GitHub]
---
之前写了EventBus的使用详解，下面来详细分析一下EventBus的源码吧。  

转载相关博客：  
EventBus源码研读:http://kymjs.com/code/2015/12/16/01  
EXECUTORSERVICE线程池讲解：http://www.cnphp6.com/archives/61093  
EventBus源码解析:http://www.trinea.cn/android/eventbus-source-analysis/  
彻底理解ThreadLocal:http://blog.csdn.net/lufeng20/article/details/24314381



# 进入源码世界
## 入口类EventBus类
从使用的流程来看，首先是EventBus#getDefault()

``` Java
	public static EventBus getDefault(){
		if(defaultInstance == null){
			synchronized(EventBus.class){
				if(defaultInstance == null){
					defaultInstance = new EventBus();
				}
			}
		}
		return defaultInstance;
	}
```
可以看出是一个单例模式，调用构造方法，再看构造方法，调用一个重载的构造方法，重载的构造方法又需要一个EventBusBuilder对象。

``` Java
	public EventBus() {
        this(DEFAULT_BUILDER);
    }
```

``` Java
	EventBus(EventBusBuilder builder) {
        ......
    }
```
## EventBusBuilder类
这个类主要用来创建EventBus对象使用。包含的属性也是EventBus的一些参数设置，build函数用于新建EventBus对象,installDefaultEventBus函数将当前设置应用于Default EventBus。

``` Java
	public class EventBusBuilder {
    	private static final ExecutorService DEFAULT_EXECUTOR_SERVICE = Executors.newCachedThreadPool();
    	boolean logSubscriberExceptions = true;
    	boolean logNoSubscriberMessages = true;
    	boolean sendSubscriberExceptionEvent = true;
    	boolean sendNoSubscriberEvent = true;
    	boolean throwSubscriberException;
    	boolean eventInheritance = true;
    	ExecutorService executorService;
    	List<Class<?>> skipMethodVerificationForClasses;

    	EventBusBuilder() {
        	this.executorService = DEFAULT_EXECUTOR_SERVICE;
    	}
    	......
   	}
```
longSubscriberExceptions :当调用事件处理函数异常时是否打印异常信息，默认为true  
longNoSubscriberMessages :没有订阅者订阅该事件时是否打印日志，默认为true  
sendSubscriberExceptionEvent :当调用事件处理函数异常时是否发送SubscriberExceptionEvent事件，如果此开关打开，订阅者可以通过  ``` Java
	public void onEvent(SubscriberExceptionEvent event{
		......
	}
```
订阅该事件进行处理，默认为true.

sendNoSubscriberEvent :如果没有订阅者，发送一条默认事件  
throwSubscriberException :当调用事件处理函数异常时是否抛出异常，默认为false,可通过
``` Java
EventBus.builder().
throwSubscriberException(true).installDefaultEventBus()
```
打开。  
eventInheritance :event的子类是否也能响应订阅者  
executorService :异步，BackGround处理方式的线程池  

通过上面可以看出，EventBus.getDefault()方法会创建一个默认的EventBusBuilder，如果我们想修改EventBusBuilder的一些配置选项，可以通过EventBus.builder()方法生成一个EventBusBuidler,然后设置自定义配置选项，最后调用installDefaultEventBus方法创建一个EventBus对象。

``` Java
	public EventBus installDefaultEventBus() {
        Class var1 = EventBus.class;
        synchronized(EventBus.class) {
            if(EventBus.defaultInstance != null) {
                throw new EventBusException("Default instance already exists. It may be only set once before it\'s used the first time to ensure consistent behavior.");
            } else {
                EventBus.defaultInstance = this.build();
                return EventBus.defaultInstance;
            }
        }
    }
```

## 三个Poster类
分析完EventBusBuilder，再来看EventBus的构造方法。

``` Java
	EventBus(EventBusBuilder builder) {
        this.currentPostingThreadState = new ThreadLocal() {
            protected EventBus.PostingThreadState initialValue() {
                return new EventBus.PostingThreadState();
            }
        };
        this.subscriptionsByEventType = new HashMap();
        this.typesBySubscriber = new HashMap();
        this.stickyEvents = new ConcurrentHashMap();
        this.mainThreadPoster = new HandlerPoster(this, Looper.getMainLooper(), 10);
        this.backgroundPoster = new BackgroundPoster(this);
        this.asyncPoster = new AsyncPoster(this);
        this.subscriberMethodFinder = new SubscriberMethodFinder(builder.skipMethodVerificationForClasses);
        this.logSubscriberExceptions = builder.logSubscriberExceptions;
        this.logNoSubscriberMessages = builder.logNoSubscriberMessages;
        this.sendSubscriberExceptionEvent = builder.sendSubscriberExceptionEvent;
        this.sendNoSubscriberEvent = builder.sendNoSubscriberEvent;
        this.throwSubscriberException = builder.throwSubscriberException;
        this.eventInheritance = builder.eventInheritance;
        this.executorService = builder.executorService;
    }
```
HandlerPoster、BackgroundPoster、AsyncPoster  
HandlerPoster:前台发送者  
BackgroundPoster:后台发送者  
AsyncPoster:后台发送者，只让队列的最后一个待订阅者去响应

每个Poster中都有一个任务队列，PendingPostQueue  
PendingPostQueue中定义了两个节点，队列的头节点和尾节点

``` Java
	private PendingPost head;
    private PendingPost tail;
```
PengdingPost类的实现：

``` Java
	final class PendingPost {
    	private static final List<PendingPost> pendingPostPool = new ArrayList();
    	Object event;
    	Subscription subscription;
    	PendingPost next;
    	......
    }
```

首先提供了一个池的设计，类似于我们的线程池，目的是为了减少对象创建的开销，当一个对象不用了，我们可以留着它，下次再需要的时候返回这个保留的而不是再去创建。

``` Java 
static PendingPost obtainPendingPost(Subscription subscription, Object event) {
        List var2 = pendingPostPool;
        synchronized(pendingPostPool) {
            int size = pendingPostPool.size();
            if(size > 0) {
                PendingPost pendingPost = (PendingPost)pendingPostPool.remove(size - 1);
                pendingPost.event = event;
                pendingPost.subscription = subscription;
                pendingPost.next = null;
                return pendingPost;
            }
        }
        return new PendingPost(event, subscription);
}
```

上面的方法会检查线程池中是否有可复用的，如果有可用的，返回可复用对象，如果没有可复用的，创建一个新的PendingPost对象。

``` Java
static void releasePendingPost(PendingPost pendingPost) {
        pendingPost.event = null;
        pendingPost.subscription = null;
        pendingPost.next = null;
        List var1 = pendingPostPool;
        synchronized(pendingPostPool) {
            if(pendingPostPool.size() < 10000) {
                pendingPostPool.add(pendingPost);
            }
        }
}
```

上述方法回收PengingPost对象，为了防止池无线增长，增加了size<1000的判断。

PendingPost分析完之后，我们看PendingPostQueue的出列和入列方式。

``` Java
	synchronized void enqueue(PendingPost pendingPost) {
        if(pendingPost == null) {
            throw new NullPointerException("null cannot be enqueued");
        } else {
            if(this.tail != null) {
                this.tail.next = pendingPost;
                this.tail = pendingPost;
            } else {
                if(this.head != null) {
                    throw new IllegalStateException("Head present, but no tail");
                }
                this.head = this.tail = pendingPost;
            }
            this.notifyAll();
        }
    }
```

首先是入列方式，tail尾的next指向当前正在入队的节点，tail指向自己（自己变成了最后一个节点），完成入队。如果是第一个元素，将head和tail都指向自己就可以了。  

``` Java
synchronized PendingPost poll() {
        PendingPost pendingPost = this.head;
        if(this.head != null) {
            this.head = this.head.next;
            if(this.head == null) {
                this.tail = null;
            }
        }
        return pendingPost;
    }
```

pendingPost为要出队的节点，将head指向head的next，完成出队，如果只有一个元素，tail置空。

PendingPostQueue再往上一级，是HandlerPost的enqueue方法。

``` Java
void enqueue(Subscription subscription, Object event) {
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        synchronized(this) {
            this.queue.enqueue(pendingPost);
            if(!this.handlerActive) {
                this.handlerActive = true;
                if(!this.sendMessage(this.obtainMessage())) {
                    throw new EventBusException("Could not send handler message");
                }
            }
        }
    }
```

根据输入参数，生成待发送对象PendingPost,然后加入队列，如果此时handlerActive是false的话，发送一条空的消息激活handler,然后是handleMessage()方法

``` Java
	public void handleMessage(Message msg) {
        boolean rescheduled = false;

        try {
            long started = SystemClock.uptimeMillis();

            long timeInMethod;
            do {
                PendingPost pendingPost = this.queue.poll();
                if(pendingPost == null) {
                    synchronized(this) {
                        pendingPost = this.queue.poll();
                        if(pendingPost == null) {
                            this.handlerActive = false;
                            return;
                        }
                    }
                }

                this.eventBus.invokeSubscriber(pendingPost);
                timeInMethod = SystemClock.uptimeMillis() - started;
            } while(timeInMethod < (long)this.maxMillisInsideHandleMessage);

            if(!this.sendMessage(this.obtainMessage())) {
                throw new EventBusException("Could not send handler message");
            }

            rescheduled = true;
        } finally {
            this.handlerActive = rescheduled;
        }

    }
```

handleMessage不停的去待发送队列queue中去取消息（timeInMehtod<maxMillisInsideHandleMessage,加判断为了防止主线程ANR），最终通过eventBus的invokeSubscriber方法发送出去，让注册了的订阅者去响应。
关于BackgroundPoster、AsyncPoster原理与HandlerPoster类似，这两个是工作在异步，实现Runnable接口，用到了ExecutorService。

## Subscribe流程
分析完EventBus的构造函数，下面看一下入口方法register().

``` Java
private synchronized void register(Object subscriber, boolean sticky, int priority) {
        List subscriberMethods = this.subscriberMethodFinder.findSubscriberMethods(subscriber.getClass());
        Iterator var5 = subscriberMethods.iterator();
        while(var5.hasNext()) {
            SubscriberMethod subscriberMethod = (SubscriberMethod)var5.next();
            this.subscribe(subscriber, subscriberMethod, sticky, priority);
        }
}
```
### SubscriberMethod类
从字面意思是订阅者方法，看看类中的方法

``` Java
	final Method method;
    final ThreadMode threadMode;
    final Class<?> eventType;
    String methodString;
    SubscriberMethod(Method method, ThreadMode threadMode, Class<?> eventType) {
        this.method = method;
        this.threadMode = threadMode;
        this.eventType = eventType;
    }
    private synchronized void checkMethodString() {
        if(this.methodString == null) {
            StringBuilder builder = new StringBuilder(64);
            builder.append(this.method.getDeclaringClass().getName());
            builder.append('#').append(this.method.getName());
            builder.append('(').append(this.eventType.getName());
            this.methodString = builder.toString();
        }
    }
```

Method：方法名  
ThreadMode:一个枚举类  
checkMehtodString()方法是为了设置变量methodString的值  
### SubscriberMethodFinder类
subscriberMethodFinder类主要是通过反射的方法来判断传入的this对象中是否有onEvent开头的方法的。

``` Java
private void filterSubscriberMethods(List<SubscriberMethod> subscriberMethods,
                                         HashMap<String, Class> eventTypesFound, StringBuilder methodKeyBuilder,
                                         Method[] methods) {
        for (Method method : methods) {
            String methodName = method.getName();
            if (methodName.startsWith(ON_EVENT_METHOD_NAME)) {
                int modifiers = method.getModifiers();
                Class<?> methodClass = method.getDeclaringClass();
                if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
                ......
                }
            }
        }
 }
```

通过对全部的方法遍历，为了效率首先做一次筛选，只关注我们的以 “onEvent” 开头的方法。

``` Java
private ThreadMode getThreadMode(Class<?> clazz, Method method, String methodName) {
        String modifierString = methodName.substring(ON_EVENT_METHOD_NAME.length());
        ThreadMode threadMode;
        if (modifierString.length() == 0) {
            threadMode = ThreadMode.PostThread;
        } else if (modifierString.equals("MainThread")) {
            threadMode = ThreadMode.MainThread;
        } else if (modifierString.equals("BackgroundThread")) {
            threadMode = ThreadMode.BackgroundThread;
        } else if (modifierString.equals("Async")) {
            threadMode = ThreadMode.Async;
        } else {
            if (!skipMethodVerificationForClasses.containsKey(clazz)) {
                throw new EventBusException("Illegal onEvent method, check for typos: " + method);
            } else {
                threadMode = null;
            }
        }
        return threadMode;
    }
```



这里我们看到，其实EventBus不仅仅支持onEvent()的回调，它还支持onEventMainThread()、onEventBackgroundThread()、onEventAsync()这三个方法的回调。  
一直到最后，我们看到这个方法把所有的方法名集合作为value，类名作为key存入了 methodCache 这个全局静态变量中。意味着，整个库在运行期间所有遍历的方法都会存在这个 map 中，而不必每次都去做耗时的反射取方法了。

``` Java
 synchronized(methodCache) {
    methodCache.put(subscriberClass, subscriberMethods1);
```

``` Java
synchronized(methodCache) {
     subscriberMethods = (List)methodCache.get(subscriberClass);
        }
```

## 事件的处理与发送subscribe()

subscribe()方法方法接收四个参数：订阅者封装的对象、响应方法名封装的对象、是否为粘滞事件、这条事件的优先级。  

``` Java
 // Must be called in synchronized block
    private void subscribe(Object subscriber, SubscriberMethod subscriberMethod, boolean sticky, int priority) {
        Class<?> eventType = subscriberMethod.eventType;
        CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
        Subscription newSubscription = new Subscription(subscriber, subscriberMethod, priority);
        if (subscriptions == null) {
            subscriptions = new CopyOnWriteArrayList<Subscription>();
            subscriptionsByEventType.put(eventType, subscriptions);
        } else {
            if (subscriptions.contains(newSubscription)) {
                throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                        + eventType);
            }
        }
        // Starting with EventBus 2.2 we enforced methods to be public (might change with annotations again)
        // subscriberMethod.method.setAccessible(true);
        int size = subscriptions.size();
        for (int i = 0; i <= size; i++) {
            if (i == size || newSubscription.priority > subscriptions.get(i).priority) {
                subscriptions.add(i, newSubscription);
                break;
            }
        }
        List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
        if (subscribedEvents == null) {
            subscribedEvents = new ArrayList<Class<?>>();
            typesBySubscriber.put(subscriber, subscribedEvents);
        }
        subscribedEvents.add(eventType);
        .......
}
```

每个订阅者是可以有多个重载的onEvent()方法的，所以这里多做了一步，将所有订阅者的响应方法保存到subscribedEvents中。

注：子事件也可以让响应父事件的 onEvent() 。这个有点绕，举个例子，订阅者的onEvent(CharSequence),如果传一个String类型的值进去，默认情况下是不会响应的，但如果我们在构建的时候设置了 eventInheritance 为 true ,那么它就会响应了。
## post()方法调用流程

``` Java
public void post(Object event) {
        PostingThreadState postingState = currentPostingThreadState.get();
        List<Object> eventQueue = postingState.eventQueue;
        eventQueue.add(event);
        if (!postingState.isPosting) {
            postingState.isMainThread = Looper.getMainLooper() == Looper.myLooper();
            postingState.isPosting = true;
            if (postingState.canceled) {
                throw new EventBusException("Internal error. Abort state was not reset");
            }
            try {
                while (!eventQueue.isEmpty()) {
                    postSingleEvent(eventQueue.remove(0), postingState);
                }
            } finally {
                postingState.isPosting = false;
                postingState.isMainThread = false;
            }
        }
}
```

post() 方法首先从 currentPostingThreadState 对象中取了一个 PostingThreadState ，我们来看看这个 currentPostingThreadState 对象的创建代码。

``` Java
private final ThreadLocal<PostingThreadState> currentPostingThreadState = new
ThreadLocal<PostingThreadState>() {
    @Override
    protected PostingThreadState initialValue() {
        return new PostingThreadState();
    }
};
```

ThreadLocal 是一个线程内部的数据存储类，通过它可以在指定的线程中存储数据，而这段数据是不会与其他线程共享的。其内部原理是通过生成一个它包裹的泛型对象的数组，在不同的线程会有不同的数组索引值，通过这样就可以做到每个线程通过 get() 方法获取的时候，取到的只能是自己线程所对应的数据。 
在 EventBus 中， ThreadLocal 所包裹的是一个 PostingThreadState 类，它仅仅是封装了一些事件发送中过程所需的数据。

``` Java
final static class PostingThreadState {
    //通过post方法参数传入的事件集合
    final List<Object> eventQueue = new ArrayList<Object>(); 
    boolean isPosting; //是否正在执行postSingleEvent()方法
    boolean isMainThread;
    Subscription subscription;
    Object event;
    boolean canceled;
    }
```

回到 post() 方法，我们看到其核心代码是这句：

``` Java
while (!eventQueue.isEmpty()) {
    postSingleEvent(eventQueue.remove(0), postingState);
}
```

次调用post()的时候都会传入一个事件，这个事件会被加入到队列。而每次执行postSingleEvent()都会从队列中取出一个事件，这样不停循环取出事件处理，直到队列全部取完。 
再看 postSingleEvent() 方法

``` Java
private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
    Class<?> eventClass = event.getClass();
    boolean subscriptionFound = false;
    if (eventInheritance) {
        //获取到eventClass所有父类的集合
        List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
        int countTypes = eventTypes.size();
        for (int h = 0; h < countTypes; h++) {
            Class<?> clazz = eventTypes.get(h);
            //左或右只要有一个为真则为真,并赋值给左
            subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
        }
    } else {
        subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
    }
    if (!subscriptionFound) {
        if (logNoSubscriberMessages) {
            Log.d(TAG, "No subscribers registered for event " + eventClass);
        }

        //参考sendNoSubscriberEvent注释
        if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
                eventClass != SubscriberExceptionEvent.class) {
            post(new NoSubscriberEvent(this, event));
        }
    }
}
```
还记得 EventBusBuild 中的 eventInheritance是做什么的吗？它表示一个子类事件能否响应父类的 onEvent() 方法。
再往下看 lookupAllEventTypes() 它通过循环和递归一起用，将一个类的父类,接口,父类的接口,父类接口的父类,全部添加到全局静态变量 eventTypes 集合中。之所以用全局静态变量的好处在于用全局静态变量只需要将那耗时又复杂的循环+递归方法执行一次就够了，下次只需要通过 key:事件类名 来判断这个事件是否以及执行过 lookupAllEventTypes() 方法。

### postSingleEventForEventType()方法

``` Java
private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
    CopyOnWriteArrayList<Subscription> subscriptions;
    synchronized (this) {
        //所有订阅了eventClass的事件集合
        subscriptions = subscriptionsByEventType.get(eventClass);
    }
    if (subscriptions != null && !subscriptions.isEmpty()) {
        //回调subscription的响应方法
        for (Subscription subscription : subscriptions) {
            postingState.event = event;
            postingState.subscription = subscription;
            boolean aborted = false;
            try {
                postToSubscription(subscription, event, postingState.isMainThread);
                aborted = postingState.canceled;
            } finally {
                postingState.event = null;
                postingState.subscription = null;
                postingState.canceled = false;
            }
            if (aborted) {
                break;
            }
        }
        return true;
    }
    return false;
}
```
获取到所有订阅了 eventClass 的事件集合，之前有讲过， subscriptionsByEventType 是一个以 key:订阅的事件 value:订阅这个事件的所有订阅者集合 的 Map 。
最后通过循环，遍历所有订阅了 eventClass 事件的订阅者，并向每一个订阅者发送事件。
看它的发送事件的方法：  
postToSubscription(subscription, event, postingState.isMainThread); 
噢，又回到了和之前 Subscribe 流程中处理粘滞事件相同的方法里————对声明不同线程模式的事件做不同的响应方法，最终都是通过invokeSubscriber()反射订阅者类中的以onEvent开头的方法。

## unregister()

``` Java
public synchronized void unregister(Object subscriber) {
    List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
    if (subscribedTypes != null) {
        for (Class<?> eventType : subscribedTypes) {
            //取消注册subscriber对eventType事件的响应
            unsubscribeByEventType(subscriber, eventType);
        }
        //当subscriber对所有事件都不响应以后,移除订阅者
        typesBySubscriber.remove(subscriber);
    }
}
```

之前讲过typesBySubscriber key:订阅者对象 value:这个订阅者订阅的事件集合，表示当前订阅者订阅了哪些事件。 
首先遍历要取消注册的订阅者订阅的每一个事件，调用unsubscribeByEventType(),从这个事件的所有订阅者集合中将要取消注册的订阅者移除。最后再以：当前订阅者为 key 全部订阅事件集合为 value 的一个 Map 的 Entry 移除，就完成了取消注册的全部过程。

## EventBus工作原理
最后我们再来从设计者的角度看一看EventBus的工作原理。

### 订阅的逻辑

1、首先是调用register()方法注册一个订阅者A。  
2、遍历这个订阅者A的全部以onEvent开头的订阅方法。  
3、将A订阅的所有事件分别作为 key，所有能响应 key 事件的订阅者的集合作为 value，存入 Map<事件，List<订阅这个事件的订阅者>>。   4、以A的类名为 key，所有 onEvent 参数类型的类名组成的集合为 value，存入 Map<订阅者，List<订阅的事件>>。  
 4.1、如果是订阅了粘滞事件的订阅者，从粘滞事件缓存区获取之前发送过的粘滞事件，响应这些粘滞事件。
### 发送事件的逻辑

1、取当前线程的发送事件封装数据，并从封装的数据中拿到发送事件的事件队列。  
2、将要发送的事件加入到事件队列中去。  
3、循环，每次发送队列中的一条事件给所有订阅了这个事件的订阅者。  
3.1、如果是子事件可以响应父事件的事件模式，需要先将这个事件的所有父类、接口、父类的接口、父类接口的父类都找到，并让订阅了这些父类信息的订阅者也都响应这条事件。

### 响应事件的逻辑

1、发送事件处理完成后会将事件交给负责响应的逻辑部分。  
2、首先判断时间的响应模式，响应模式分为四种：  
PostThread 在哪个线程调用的post()方法，就在哪个线程执行响应方法。  
MainThread 无论是在哪个线程调用的post()方法，最终都在主线程执行响应方法。  
BackgroundThread 无论是在哪个线程调用的post()方法，最终都在后台线程执行响应方法。(串行执行，一次只执行一个任务，其他任务在队列中处于等待状态)  
Async 无论是在哪个线程调用的post()方法，最终都在后台线程执行响应方法。(并行执行，只要有任务就开一个线程让他执行)

### 取消注册的逻辑

1、首先是调用unregister()方法拿到要取消注册的订阅者B。   
2、从这个类订阅的时候存入的 Map<订阅者，List<订阅的事件>> 中，拿到这个类的订阅事件集合。   
3、遍历订阅时间集合，在注册的时候存入的 Map<事件，List<订阅这个事件的订阅者>> 中将对应订阅事件的订阅者集合中的这个订阅者移除。  
   4、将步骤2中的 Map<订阅者，List<订阅的事件>> 中这个订阅者相关的 Entry 移除。