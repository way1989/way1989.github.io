---
date: 2016-08-18 11:16:24
title: EventBus # 这是标题
description: EventBus使用简介
categories:   # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
 - 博客
 - EventBus
tags: # 这里写的标签会自动汇集到 tags 页面上
 - 博客
---

EventBus
========
EventBus is a publish/subscribe event bus optimized for Android.<br/>
<img src="https://raw.githubusercontent.com/way1989/way1989.github.io/master/uploads/images/eventbus/EventBus-Publish-Subscribe.png" width="500" height="187"/>

## EventBus介绍

 * simplifies the communication between components
    * decouples event senders and receivers  
    * performs well with Activities, Fragments, and background threads
    * avoids complex and error-prone dependencies and life cycle issues
 * makes your code simpler
 * is fast
 * is tiny (~50k jar)
 * is proven in practice by apps with 100,000,000+ installs
 * has advanced features like delivery threads, subscriber priorities, etc.

## 添加EventBus支持

Gradle:
```
gradle
compile 'org.greenrobot:eventbus:3.0.0'
```
[download EventBus from Maven Central](http://search.maven.org/#search%7Cga%7C1%7Cg%3A%22de.greenrobot%22%20AND%20a%3A%22eventbus%22)

## EventBus 使用示例

### 定义事件

```java  
  public class LoadEvent {
}
```

### onCreate中注册事件
```java
    EventBus.getDefault().register(this);
```

### onDestroy中取消注册
```java
  EventBus.getDefault().unregister(this);
```

### 订阅事件

```java
    @Subscribe(threadMode = ThreadMode.MAIN, priority = 0, sticky = false)
    public void onLoadEvent(LoadEvent event) {
        Log.d(TAG, "onLoadEvent");
    }
```

  * threadMode
    * POSTING 在调用post所在的线程执行回调，直接运行。
    * MAIN 在主线程回调，如果post所在线程为主线程则直接执行，否则则通过mainThreadPoster来调度。
    * BACKGROUND 如果post所在线程为非UI线程则直接执行，否则则通过backgroundPoster来调度，这里只适合执行时间比较短的任务。
    * ASYNC（交给线程池来管理）：直接通过asyncPoster调度。
  * sticky 是否监听黏性事件      
      >If true, delivers the most recent sticky event (posted with EventBus#postSticky(Object)) to this subscriber (if event available).

### 发布事件
  ```java
   EventBus.getDefault().post(new LoadEvent());
  ```

### 完整示例

```java
    public class MainActivity extends AppCompatActivity {
        public static final String TAG = "MainActivity";

        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.activity_main);

            EventBus.getDefault().register(this);

            Button start = (Button) findViewById(R.id.start_load);
            start.setOnClickListener(new View.OnClickListener() {
                @Override
                public void onClick(View v) {
                    EventBus.getDefault().post(new LoadEvent());
                }
            });
        }

        @Override
        protected void onDestroy() {
            super.onDestroy();
            EventBus.getDefault().unregister(this);
        }

        @Subscribe(threadMode = ThreadMode.MAIN, priority = 0, sticky = false)
        public void onLoadEvent(LoadEvent event) {
            Log.d(TAG, "onLoadEvent");
        }

        public static class LoadEvent {

        }
    }
```

## EventBus的注解生成索引

### 项目的根目录build.gradle引入apt编译插件

```gradle
  classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'
```
### app的build.gradle 应用apt插件，并设置apt生成的索引的包名和类名

```gradle
  apply plugin: 'com.neenbedankt.android-apt'
  apt {
      arguments {
          eventBusIndex "com.android.gallery3d.MySubscriberInfoIndex"
      }
  }
```
### app的dependencies中引入EventBusAnnotationProcessor：

```gradle
 apt 'org.greenrobot:eventbus-annotation-processor:3.0.1'
```
### 把注解生成的索引添加到EventBus默认的单例中，注意installDefaultEventBus这个方法只能调用一次

```java
EventBus.builder().addIndex(new MySubscriberInfoIndex()).installDefaultEventBus();  
```

## EventBus原理浅析

### EventBus.getDefault()方法

```java
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

### EventBus.getDefault().register(this)方法

  <img src="https://raw.githubusercontent.com/way1989/way1989.github.io/master/uploads/images/eventbus//register.png"/>


```java
  public void register(Object subscriber) {
      Class<?> subscriberClass = subscriber.getClass();
      List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
      synchronized (this) {
          for (SubscriberMethod subscriberMethod : subscriberMethods) {
              subscribe(subscriber, subscriberMethod);
          }
      }
  }  
```


```java
  private final Map<Class<?>, CopyOnWriteArrayList<Subscription>> subscriptionsByEventType;

  private final Map<Object, List<Class<?>>> typesBySubscriber;  

  private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
      Class<?> eventType = subscriberMethod.eventType;
      Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
      CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
      if (subscriptions == null) {
          subscriptions = new CopyOnWriteArrayList<>();
          subscriptionsByEventType.put(eventType, subscriptions);
      } else {
          if (subscriptions.contains(newSubscription)) {
              throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                      + eventType);
          }
      }

      int size = subscriptions.size();
      for (int i = 0; i <= size; i++) {
          if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
              subscriptions.add(i, newSubscription);
              break;
          }
      }

      List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
      if (subscribedEvents == null) {
          subscribedEvents = new ArrayList<>();
          typesBySubscriber.put(subscriber, subscribedEvents);
      }
      subscribedEvents.add(eventType);

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
              Object stickyEvent = stickyEvents.get(eventType);
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
          postToSubscription(newSubscription, stickyEvent, Looper.getMainLooper() == Looper.myLooper());
      }
  }  
```


```java
  private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
      switch (subscription.subscriberMethod.threadMode) {
          case POSTING:
              invokeSubscriber(subscription, event);
              break;
          case MAIN:
              if (isMainThread) {
                  invokeSubscriber(subscription, event);
              } else {
                  mainThreadPoster.enqueue(subscription, event);
              }
              break;
          case BACKGROUND:
              if (isMainThread) {
                  backgroundPoster.enqueue(subscription, event);
              } else {
                  invokeSubscriber(subscription, event);
              }
              break;
          case ASYNC:
              asyncPoster.enqueue(subscription, event);
              break;
          default:
              throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
      }
  }  
```

### EventBus.getDefault().post(new LoadEvent());

  <img src="https://raw.githubusercontent.com/way1989/way1989.github.io/master/uploads/images/eventbus//post.png"/>


```java
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

```java
  private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
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
              Log.d(TAG, "No subscribers registered for event " + eventClass);
          }
          if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
                  eventClass != SubscriberExceptionEvent.class) {
              post(new NoSubscriberEvent(this, event));
          }
      }
  }  
```


```java
  private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
    CopyOnWriteArrayList<Subscription> subscriptions;
    synchronized (this) {
        subscriptions = subscriptionsByEventType.get(eventClass);
    }
    if (subscriptions != null && !subscriptions.isEmpty()) {
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


```java
  private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
      switch (subscription.subscriberMethod.threadMode) {
          case POSTING:
              invokeSubscriber(subscription, event);
              break;
          case MAIN:
              if (isMainThread) {
                  invokeSubscriber(subscription, event);
              } else {
                  mainThreadPoster.enqueue(subscription, event);
              }
              break;
          case BACKGROUND:
              if (isMainThread) {
                  backgroundPoster.enqueue(subscription, event);
              } else {
                  invokeSubscriber(subscription, event);
              }
              break;
          case ASYNC:
              asyncPoster.enqueue(subscription, event);
              break;
          default:
              throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
      }
  }  
```


```java
  void invokeSubscriber(Subscription subscription, Object event) {
      try {
          subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
      } catch (InvocationTargetException e) {
          handleSubscriberException(subscription, event, e.getCause());
      } catch (IllegalAccessException e) {
          throw new IllegalStateException("Unexpected exception", e);
      }
  }  
```


### EventBus.getDefault().unregister(this);

  <img src="https://raw.githubusercontent.com/way1989/way1989.github.io/master/images/unregister.png"/>


```java
  private final Map<Object, List<Class<?>>> typesBySubscriber;

  public synchronized void unregister(Object subscriber) {
      List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
      if (subscribedTypes != null) {
          for (Class<?> eventType : subscribedTypes) {
              unsubscribeByEventType(subscriber, eventType);
          }
          typesBySubscriber.remove(subscriber);
      } else {
          Log.w(TAG, "Subscriber to unregister was not registered before: " + subscriber.getClass());
      }
  }  
```


```java
  private final Map<Class<?>, CopyOnWriteArrayList<Subscription>> subscriptionsByEventType;

  private void unsubscribeByEventType(Object subscriber, Class<?> eventType) {
      List<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
      if (subscriptions != null) {
          int size = subscriptions.size();
          for (int i = 0; i < size; i++) {
              Subscription subscription = subscriptions.get(i);
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

## EventBus缺点

* 耦合性太低
* 订阅的事件是利用反射的机制执行的
* 不能跨进程


## 用RxJava实现简易RxBus

### 实现RxBus

```java
  public class RxBus {
      private static volatile RxBus sInstance;
      private final Subject subject;

      private RxBus() {
          subject = new SerializedSubject<>(PublishSubject.create());
      }

      public static RxBus getInstance() {
          if (sInstance == null) {
              synchronized (RxBus.class) {
                  if (sInstance == null) {
                      sInstance = new RxBus();
                  }
              }
          }
          return sInstance;
      }

      public void post(Object event) {
          subject.onNext(event);
      }

      public <T> Observable<T> toObservable(Class<T> eventType) {
          return subject.ofType(eventType);
      }
  }  
```

### 订阅事件

```java
  RxBus.getInstance().toObservable(ForceUpdateEvent.class)
          .observeOn(AndroidSchedulers.mainThread())
          .compose(this.<ForceUpdateEvent>bindUntilEvent(ActivityEvent.DESTROY))
          .subscribe(new Action1<ForceUpdateEvent>() {
              @Override
              public void call(ForceUpdateEvent forceUpdateEvent) {
                  //do some thing
              }
          });  
```

### 发布事件

```java
  RxBus.getInstance().post(new ForceUpdateEvent(parameter));  
```


```java
  public class ForceUpdateEvent {
  }  
```

## 参考链接

* [EventBus github](https://github.com/greenrobot/EventBus)
* [Bugly干货分享】老司机教你 “飙” EventBus 3](http://www.cnblogs.com/bugly/p/5475034.html)
* [Android EventBus源码解析 带你深入理解EventBus](http://blog.csdn.net/lmj623565791/article/details/40920453)
* [用RxJava实现事件总线(Event Bus)](http://www.jianshu.com/p/ca090f6e2fe2)
