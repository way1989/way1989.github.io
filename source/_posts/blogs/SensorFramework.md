---
date: 2016-09-15 18:00:24
title: Android Sensor框架Framework层解读
description: Android Sensor框架Framework层解读
categories:   # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
 - 博客
 - Sensor
tags: # 这里写的标签会自动汇集到 tags 页面上
 - 博客
 - Sensor
---

## Sensor整体架构：

<img src="https://raw.githubusercontent.com/way1989/way1989.github.io/master/uploads/images/sensorFramework/1.png"/>

### 整体架构说明:

1. 黄色部分表示硬件，它要挂在I2C总线上    
2. 红色部分表示驱动，驱动注册到Kernel的Input Subsystem上，然后通过Event Device把Sensor数据传到HAL层，准确说是HAL从Event读
3. 绿色部分表示动态库，它封装了整个Sensor的IPC机制，如SensorManager是客户端，SensorService是服务端，而HAL部分是封装了服务端对Kernel的直接访问
4. 蓝色部分就是我们的Framework和Application了，JNI负责访问Sensor的客户端，而Application就是具体的应用程序，用来接收Sensor返回的数据，并处理实现对应的UI效果，如屏幕旋转，打电话时灭屏，自动调接背光（这三个功能的具体实现会在以后分析）

### Sensor总体调用关系图

- 本节主要解读Android的Framework层框架。
- Sensor 框架分为三个层次，客户度、服务端、HAL层，服务端负责从HAL读取数据，并将数据写到管道中，客户端通过管道读取服务端数据。

<img src="https://raw.githubusercontent.com/way1989/way1989.github.io/master/uploads/images/sensorFramework/2.png"/>

### 客户端主要类

- `SensorManager.java`：从android4.1开始，把SensorManager定义为一个抽象类，定义了一些主要的方法，该类主要是应用层直接使用的类，提供给应用层的接口
- `SystemSensorManager.java`：继承于SensorManager，客户端消息处理的实体，应用程序通过获取其实例，并注册监听接口，获取sensor数据。
- `sensorEventListener`接口：用于注册监听的接口
- `sensorThread`：是SystemSensorManager的一个内部类，开启一个新线程负责读取读取sensor数据，当注册了sensorEventListener接口的时候才会启动线程
- `android_hardware_SensorManager.cpp`：负责与java层通信的JNI接口
- `SensorManager.cpp`：sensor在Native层的客户端，负责与服务端SensorService.cpp的通信
- `SenorEventQueue.cpp`：消息队列

### 服务端主要类

- `SensorService.cpp`：服务端数据处理中心
- `SensorEventConnection`：从BnSensorEventConnection继承来，实现接口ISensorEventConnection的一些方法，ISensorEventConnection在SensorEventQueue会保存一个指针，指向调用服务接口创建的SensorEventConnection对象
- `Bittube.cpp`：在这个类中创建了管道，用于服务端与客户端读写数据
- `SensorDevice`：负责与HAL读取数据

### HAL层
- `Sensor.h`是google为Sensor定义的Hal接口，单独提出去


## 客户端读取数据

### 调用时序图

<img src="https://raw.githubusercontent.com/way1989/way1989.github.io/master/uploads/images/sensorFramework/3.jpg"/>

### apk注册监听器

- Activity实现了SensorEventListener接口。在onCreate方法中，获取SystemSensorManager，并获取到加速传感器的Sensor；在onResume方法中调用SystemSensorManager，registerListenerImpl注册监听器；当Sensor数据有改变的时候将会回调onSensorChanged方法。

``` java
SensorManager  mSensorManager =
 (SensorManager)getSystemService(SENSOR_SERVICE);
 Sensor   mAccelerometer =
 mSensorManager.getDefaultSensor(Sensor.TYPE_ACCELEROMETER);

    protected void onResume() {
           super.onResume();
          mSensorManager. registerListenerImpl (this, mAccelerometer,
     SensorManager.SENSOR_DELAY_NORMAL);
     }
     protected void onPause() {
           super.onPause();
         mSensorManager.unregisterListener(this);
     }

public interface SensorEventListener {
    public void onSensorChanged(SensorEvent event);
    public void onAccuracyChanged(Sensor sensor, int accuracy);   
}
```

### 初始化SystemSensorManager

- 系统开机启动的时候，会创建SystemSensorManager的实例，在其构造函数中，主要做了四件事情：

    1. 初始化JNI：调用JNI函数nativeClassInit()进行初始化
    2. 初始化Sensor列表：调用JNI函数sensors_module_init，对Sensor模块进行初始化。创建了native层SensorManager的实例。
    3. 获取Sensor列表：调用JNI函数sensors_module_get_next_sensor()获取Sensor，并存在sHandleToSensor列表中
    4. 构造SensorThread类：构造线程的类函数，并没有启动线程，当有应用注册的时候才会启动线程


``` java
public SystemSensorManager(Context context,Looper mainLooper) {
        mMainLooper = mainLooper;       
              mContext = context;

        synchronized(sListeners) {
            if (!sSensorModuleInitialized) {
                sSensorModuleInitialized = true;

                nativeClassInit();

                // initialize the sensor list
                sensors_module_init();
                final ArrayList<Sensor> fullList = sFullSensorsList;
                int i = 0;
                do {
                    Sensor sensor = new Sensor();
                    i = sensors_module_get_next_sensor(sensor, i);

                    if (i>=0) {
                        //Log.d(TAG, "found sensor: " + sensor.getName() +
                        //        ", handle=" + sensor.getHandle());
                        fullList.add(sensor);
                        sHandleToSensor.append(sensor.getHandle(), sensor);
                    }
                } while (i>0);

                sPool = new SensorEventPool( sFullSensorsList.size()*2 );
                sSensorThread = new SensorThread();
            }
        }
    }
```

### 启动SensorThread线程读取消息队列中数据

- 当有应用程序调用registerListenerImpl()方法注册监听的时候，会调用SensorThread.startLoacked()启动线程。线程只会启动一次，并调用enableSensorLocked()接口对指定的sensor使能，并设置采样时间。　　SensorThreadRunnable实现了Runnable接口，在SensorThread类中被启动。

``` java
protected boolean registerListenerImpl(SensorEventListener listener, Sensor sensor,
            int delay, Handler handler) {

        synchronized (sListeners) {
            ListenerDelegate l = null;
            for (ListenerDelegate i : sListeners) {
                if (i.getListener() == listener) {
                    l = i;
                }
            }
            …….
            // if we don't find it, add it to the list
            if (l == null) {
                l = new ListenerDelegate(listener, sensor, handler);
                sListeners.add(l);
                  ……
                    if (sSensorThread.startLocked()) {
                        if (!enableSensorLocked(sensor, delay)) {
                          …….
                        }
                 ……
            } else if (!l.hasSensor(sensor)) {
                l.addSensor(sensor);
                if (!enableSensorLocked(sensor, delay)) {
                    ……
                }
            }
        }
        return result;
    }
```

- 在open函数中调用JNI函数sensors_create_queue()来创建消息队列,然后调用SensorManager. createEventQueue()创建。在startLocked函数中启动新的线程后，做了一个while的等待while (mSensorsReady == false)，只有当mSensorsReady等于true的时候，才会执行enableSensorLocked()函数对sensor使能。而mSensorsReady变量，是在open()调用创建消息队列成功之后才会true，所以认为，三个功能调用顺序是如下：
    1. 调用open函数创建消息队列
    2. 调用enableSensorLocked()函数对sensor使能
    3. 调用sensors_data_poll从消息队列中读取数据，而且是在while (true)循环里一直读取

``` java
boolean startLocked() {
            try {
                if (mThread == null) {
                    SensorThreadRunnable runnable = new SensorThreadRunnable();
                    Thread thread = new Thread(runnable, SensorThread.class.getName());
                    thread.start();
                    synchronized (runnable) {  //队列创建成功,线程同步
                        while (mSensorsReady == false) {
                            runnable.wait();
                        }
                    }

        }
private class SensorThreadRunnable implements Runnable {
            SensorThreadRunnable() {
            }
            private boolean open() {
                sQueue = sensors_create_queue();
                return true;
            }
            public void run() {
                …….
                if (!open()) {
                    return;
                }
                synchronized (this) {
                    mSensorsReady = true;
                    this.notify();
                }
                while (true) {
                    final int sensor = sensors_data_poll(sQueue, values, status, timestamp);
                    …….
            }
        }
    }
```

## 服务端实现

### 调用时序图

<img src="https://raw.githubusercontent.com/way1989/way1989.github.io/master/uploads/images/sensorFramework/4.png"/>

### 启动SensorService服务

- 在SystemServer进程中的main函数中，通过JNI调用，调用到`com_android_server_SystemServer.cpp`的`android_server_SystemServer_init1()`方法，该方法又调用`system_init.cpp中的system_init()`，在这里创建了SensorService的实例。

``` cpp
extern "C" status_t system_init()
{
    ……
    property_get("system_init.startsensorservice", propBuf, "1");
    if (strcmp(propBuf, "1") == 0) {
        // Start the sensor service
        SensorService::instantiate();
    }
    …..
    return NO_ERROR;
}
```

### SensorService初始化

- SensorService创建完之后，将会调用SensorService::onFirstRef()方法，在该方法中完成初始化工作。首先获取SensorDevice实例，在其构造函数中，完成了对Sensor模块HAL的初始化：

``` cpp
SensorDevice::SensorDevice()
    :  mSensorDevice(0),
       mSensorModule(0)
{
    status_t err = hw_get_module(SENSORS_HARDWARE_MODULE_ID,
            (hw_module_t const**)&mSensorModule);

    if (mSensorModule) {
        err = sensors_open(&mSensorModule->common, &mSensorDevice);

        ALOGE_IF(err, "couldn't open device for module %s (%s)",
                SENSORS_HARDWARE_MODULE_ID, strerror(-err));

        if (mSensorDevice) {
            sensor_t const* list;
            ssize_t count = mSensorModule->get_sensors_list(mSensorModule, &list);
            mActivationCount.setCapacity(count);
            Info model;
            for (size_t i=0 ; i<size_t(count) ; i++) {
                mActivationCount.add(list[i].handle, model);
                mSensorDevice->activate(mSensorDevice, list[i].handle, 0);
            }
        }
    }
}
```

- 这里主要做了三个工作：
    1. 调用HAL层的hw_get_modele()方法，加载Sensor模块so文件
    2. 调用sensor.h的sensors_open方法打开设备
    3. 调用sensors_poll_device_t->activate()对Sensor模块使能

- 再看看SensorService::onFirstRef()方法:

``` cpp
void SensorService::onFirstRef()
{
    SensorDevice& dev(SensorDevice::getInstance());

    if (dev.initCheck() == NO_ERROR) {
        sensor_t const* list;
        ssize_t count = dev.getSensorList(&list);
        if (count > 0) {
            ……
            for (ssize_t i=0 ; i<count ; i++) {
                registerSensor( new HardwareSensor(list[i]) );
                ……
            }

            // it's safe to instantiate the SensorFusion object here
            // (it wants to be instantiated after h/w sensors have been
            // registered)
            const SensorFusion& fusion(SensorFusion::getInstance());

            if (hasGyro) {
               ……
            }
            ……
            run("SensorService", PRIORITY_URGENT_DISPLAY);
            mInitCheck = NO_ERROR;
        }
    }
}
```

- 在这个方法中，主要做了4件事情：
    1. 创建SensorDevice实例
    2. 获取Sensor列表
    3. 调用SensorDevice.getSensorList(),获取Sensor模块所有传感器列表
    4. 为每个传感器注册监听器

- `registerSensor(new HardwareSensor(list[i]))`;

``` cpp
void SensorService::registerSensor(SensorInterface* s)
{
    sensors_event_t event;
    memset(&event, 0, sizeof(event));

    const Sensor sensor(s->getSensor());
    // 添加到Sensor列表，给客户端使用
    mSensorList.add(sensor);
    // add to our handle->SensorInterface mapping
    mSensorMap.add(sensor.getHandle(), s);
    // create an entry in the mLastEventSeen array
    mLastEventSeen.add(sensor.getHandle(), event);
}
```

- HardwareSensor实现了SensorInterface接口。启动线程读取数据，调用run方法启动新线程，将调用SensorService::threadLoop()方法。

### 在新的线程中读取HAL层数据

- SensorService实现了Thread类，当在onFirstRef中调用run方法后，将在新的线程中调用`SensorService::threadLoop()`方法。在while循环中一直读取HAL层数据，再调用`SensorEventConnection->sendEvents`将数据写到管道中。

``` cpp
bool SensorService::threadLoop()
{
    ……
    do {
        count = device.poll(buffer, numEventMax);

        recordLastValue(buffer, count);
        ……

        // send our events to clients...
        const SortedVector< wp<SensorEventConnection> > activeConnections(
                getActiveConnections());
        size_t numConnections = activeConnections.size();
        for (size_t i=0 ; i<numConnections ; i++) {
            sp<SensorEventConnection> connection(
                    activeConnections[i].promote());
            if (connection != 0) {
                connection->sendEvents(buffer, count, scratch);
            }
        }
    } while (count >= 0 || Thread::exitPending());
    return false;
}
```

## 客户端与服务端通信

### 数据传送

- 客户端与服务端通信的状态图：

<img src="https://raw.githubusercontent.com/way1989/way1989.github.io/master/uploads/images/sensorFramework/5.png"/>

- 客户端服务端线程，在图中可以看到有两个线程：
    1. 一个是服务端的一个线程，这个线程负责源源不断的从HAL读取数据。
    2. 另一个是客户端的一个线程，客户端线程负责从消息队列中读数据。
- 创建消息队列，客户端可以创建多个消息队列，一个消息队列对应有一个与服务器通信的连接接口
- 创建连接接口，服务端与客户端沟通的桥梁，服务端读取到HAL层数据后，会扫面有多少个与客户端连接的接口，然后往每个接口的管道中写数据
- 创建管道，每一个连接接口都有对应的一个管道。
- 上面是设计者设计数据传送的原理，但是目前Android上面的数据传送不能完全按照上面的理解。因为在实际使用中，消息队列只会创建一个，也就是说客户端与服务端之间的通信只有一个连接接口，只有一个管道传数据。那么数据的形式是怎么从HAL层传到JAVA层的呢？其实数据是以一个结构体sensors_event_t的形式从HAL层传到JNI层。看看HAL的sensors_event_t结构体：

``` cpp
typedef struct sensors_event_t {
    int32_t version;
    int32_t sensor;            //标识符
    int32_t type;             //传感器类型
    int32_t reserved0;
    int64_t timestamp;        //时间戳
    union {
        float           data[16];
        sensors_vec_t   acceleration;   //加速度
        sensors_vec_t   magnetic;      //磁矢量
        sensors_vec_t   orientation;     //方向
        sensors_vec_t   gyro;          //陀螺仪
        float           temperature;     //温度
        float           distance;        //距离
        float           light;           //光照
        float           pressure;         //压力
        float           relative_humidity;  //相对湿度
    };
    uint32_t        reserved1[4];
} sensors_event_t;
```

- 在JNI层有一个ASensorEvent结构体与sensors_event_t向对应，frameworks/native/include/android/sensor.h：

``` cpp
typedef struct ASensorEvent {
    int32_t version;
    int32_t sensor;
    int32_t type;
    int32_t reserved0;
    int64_t timestamp;
    union {
        float           data[16];
        ASensorVector   vector;
        ASensorVector   acceleration;
        ASensorVector   magnetic;
        float           temperature;
        float           distance;
        float           light;
        float           pressure;
    };
    int32_t reserved1[4];
} ASensorEvent;
```

## 交互调用时序图

<img src="https://raw.githubusercontent.com/way1989/way1989.github.io/master/uploads/images/sensorFramework/6.jpg"/>

- 经过前面的介绍，现在知道了客户端实现的方式及服务端的实现，但是没有具体讲到它们是如何进行通信的，现在看看客户端与服务端之间的通信。
- 主要涉及的是进程间通信，有IBind和管道通信。
- 客户端通过IBind通信获取到服务端的远程调用，然后通过管道进行sensor数据的传输。

### 服务端

- native层实现了sensor服务的核心实现，Sensor服务的主要流程的实现在sensorservice类中，下面重点分析下这个类的流程。

``` cpp
class SensorService :
        public BinderService<SensorService>,
        public BnSensorServer,
        protected Thread
```

- 看看sensorService继承的类:继承BinderService<SensorService>这个模板类添加到系统服务,用于Ibinder进程间通信。

``` cpp
template<typename SERVICE>
class BinderService
{
public:
    static status_t publish() {
        sp<IServiceManager> sm(defaultServiceManager());
        return sm->addService(String16(SERVICE::getServiceName()), new SERVICE());
    }

    static void publishAndJoinThreadPool() {
        sp<ProcessState> proc(ProcessState::self());
        sp<IServiceManager> sm(defaultServiceManager());
        sm->addService(String16(SERVICE::getServiceName()), new SERVICE());
        ProcessState::self()->startThreadPool();
        IPCThreadState::self()->joinThreadPool();
    }

    static void instantiate() { publish(); }
};
}; // namespace android
```

- 在前面的介绍中，SensorService服务的实例是在System_init.cpp中调用SensorService::instantiate()创建的，即调用了上面的instantiate()方法，接着调用了publish(),在该方法中，我们看到了new SensorService的实例，并且调用了defaultServiceManager::addService()将Sensor服务添加到了系统服务管理中，客户端可以通过defaultServiceManager:getService()获取到Sensor服务的实例。

- 继承BnSensorServer这个是sensor服务抽象接口类提供给客户端调用：

``` cpp
class Sensor;
class ISensorEventConnection;

class ISensorServer : public IInterface
{
public:
    DECLARE_META_INTERFACE(SensorServer);
    //获取Sensor列表
virtual Vector<Sensor> getSensorList() = 0;
//创建一个连接的接口,这些都是提供给客户端的抽象接口,服务端实例化时候必须实现
    virtual sp<ISensorEventConnection> createSensorEventConnection() = 0;
};
class BnSensorServer : public BnInterface<ISensorServer>
{
public:
    //传输打包数据的通讯接口,在BnSensorServer被实现
    virtual status_t    onTransact( uint32_t code,
                                    const Parcel& data,
                                    Parcel* reply,
                                    uint32_t flags = 0);
};
}; // namespace android
```

- ISensorServer接口提供了两个抽象方法给客户端调用，关键在于createSensorEventConnection()方法，该在服务端被实现，在客户端被调用，并返回一个SensorEventConnection的实例，创建连接，客户端拿到SensorEventConnection实例之后，可以对sensor进行通信操作，仅仅作为通信的接口而已，它并没有用来传送Sensor数据，因为Sensor数据量比较大，IBind实现比较困难。真正实现Sensor数据传送的是管道，在创建SensorEventConnection实例中，创建了BitTube对象，里面创建了管道，用于客户端与服务端的通信。

### 客户端

- 时序图：

<img src="https://raw.githubusercontent.com/way1989/way1989.github.io/master/uploads/images/sensorFramework/7.png"/>

- 客户端主要在SensorManager.cpp中创建消息队列:

``` cpp
class ISensorEventConnection;
class Sensor;
class Looper;

// ----------------------------------------------------------------------------

class SensorEventQueue : public ASensorEventQueue, public RefBase
{
public:
            SensorEventQueue(const sp<ISensorEventConnection>& connection);
    virtual ~SensorEventQueue();
    virtual void onFirstRef();
    //获取管道句柄
    int getFd() const;
    //向管道写数据
    static ssize_t write(const sp<BitTube>& tube,
            ASensorEvent const* events, size_t numEvents);
    //向管道读数据
    ssize_t read(ASensorEvent* events, size_t numEvents);

    status_t waitForEvent() const;
    status_t wake() const;
    //使能Sensor传感器
    status_t enableSensor(Sensor const* sensor) const;
    status_t disableSensor(Sensor const* sensor) const;
    status_t setEventRate(Sensor const* sensor, nsecs_t ns) const;

    // these are here only to support SensorManager.java
    status_t enableSensor(int32_t handle, int32_t us) const;
    status_t disableSensor(int32_t handle) const;

private:
sp<Looper> getLooper() const;
//连接接口，在SensorService中创建的
sp<ISensorEventConnection> mSensorEventConnection;
//管道指针
    sp<BitTube> mSensorChannel;
    mutable Mutex mLock;
    mutable sp<Looper> mLooper;
};
```

- SensorEventQueue类作为消息队列，作用非常重要，在创建其实例的时候，传入了SensorEventConnection的实例，SensorEventConnection继承于ISensorEventConnection。SensorEventConnection其实是客户端调用SensorService的createSensorEventConnection()方法创建的，它是客户端与服务端沟通的桥梁，通过这个桥梁，可以完成一下任务：
    1. 获取管道的句柄
    2. 往管道读写数据
    3. 通知服务端对Sensor使能

## 流程解析

### 客户端获取SensorService服务实例

- 客户端初始化的时候，即SystemSensorManager的构造函数中，通过JNI调用，创建native层SensorManager的实例，然后调用SensorManager::assertStateLocked()方法做一些初始化的动作。

``` cpp
status_t SensorManager::assertStateLocked() const {
    if (mSensorServer == NULL) {
        // try for one second
        const String16 name("sensorservice");
        ……
            status_t err = getService(name, &mSensorServer);
        ……
        mSensors = mSensorServer->getSensorList();
        size_t count = mSensors.size();
        mSensorList = (Sensor const**)malloc(count * sizeof(Sensor*));
        for (size_t i=0 ; i<count ; i++) {
            mSensorList[i] = mSensors.array() + i;
        }
    }
    return NO_ERROR;
}
```

- 前面我们讲到过，SensorService的创建的时候调用了defaultServiceManager:getService()将服务添加到了系统服务管理中。现在我们又调用defaultServiceManager::geService()获取到SensorService服务的实例。在通过IBind通信，就可以获取到Sensor列表，所以在客户端初始化的时候，做了两件事情：
    1. 获取SensorService实例引用
    2. 获取Sensor传感器列表

### 注册SensorLisenter

- 时序图：

<img src="https://raw.githubusercontent.com/way1989/way1989.github.io/master/uploads/images/sensorFramework/8.png"/>

- `new ListenerDelegate(SensorEventListener listener, Sensor sensor, Handler handler)`在这个构造函数中会创建一个Handler，它会在获取到Sensor数据的时候被调用。

``` java
mHandler = new Handler(looper) {  
                @Override  
                public void handleMessage(Message msg) {  
                    final SensorEvent t = (SensorEvent)msg.obj;  
                    final int handle = t.sensor.getHandle();  

                    switch (t.sensor.getType()) {  
                        // Only report accuracy for sensors that support it.  
                        case Sensor.TYPE_MAGNETIC_FIELD:  
                        case Sensor.TYPE_ORIENTATION:  
                            // call onAccuracyChanged() only if the value changes  
                            final int accuracy = mSensorAccuracies.get(handle);  
                            if ((t.accuracy >= 0) && (accuracy != t.accuracy)) {  
                                mSensorAccuracies.put(handle, t.accuracy);  
                                mSensorEventListener.onAccuracyChanged(t.sensor, t.accuracy);  
                            }  
                            break;  
                        default:  
                            // For other sensors, just report the accuracy once  
                            if (mFirstEvent.get(handle) == false) {  
                                mFirstEvent.put(handle, true);  
                                mSensorEventListener.onAccuracyChanged(  
                                        t.sensor, SENSOR_STATUS_ACCURACY_HIGH);  
                            }  
                            break;  
                    }  

                    mSensorEventListener.onSensorChanged(t);  
                    sPool.returnToPool(t);  
                }  
            };  
```

### 创建消息队列

- 时序图：

<img src="https://raw.githubusercontent.com/way1989/way1989.github.io/master/uploads/images/sensorFramework/9.png"/>

- 当客户端第一次注册监听器的时候，就需要创建一个消息队列，也就是说，__android在目前的实现中，只创建了一个消息队列，一个消息队列中有一个管道，用于服务端与客户断传送Sensor数据__。

- 在SensorManager.cpp中的createEventQueue方法创建消息队列：

``` cpp
sp<SensorEventQueue> SensorManager::createEventQueue()
{
    sp<SensorEventQueue> queue;
    Mutex::Autolock _l(mLock);
while (assertStateLocked() == NO_ERROR) {
    //创建连接接口
        sp<ISensorEventConnection> connection =
                mSensorServer->createSensorEventConnection();
        if (connection == NULL) {
            // SensorService just died.
            LOGE("createEventQueue: connection is NULL. SensorService died.");
            continue;
        }
//创建消息队列
        queue = new SensorEventQueue(connection);
        break;
    }
    return queue;
}
```

- 客户端与服务器创建一个`SensorEventConnection`连接接口，__而一个消息队列中包含一个连接接口__。创建连接接口：

``` cpp
sp<ISensorEventConnection> SensorService::createSensorEventConnection()
{
    sp<SensorEventConnection> result(new SensorEventConnection(this));
    return result;
}
SensorService::SensorEventConnection::SensorEventConnection(
        const sp<SensorService>& service)
    : mService(service), mChannel(new BitTube ())
{
}
```

- 关键在于BitTube，在构造函数中创建了管道：

``` cpp
BitTube::BitTube()
    : mSendFd(-1), mReceiveFd(-1)
{
    int sockets[2];
    if (socketpair(AF_UNIX, SOCK_SEQPACKET, 0, sockets) == 0) {
        int size = SOCKET_BUFFER_SIZE;
        setsockopt(sockets[0], SOL_SOCKET, SO_SNDBUF, &size, sizeof(size));
        setsockopt(sockets[0], SOL_SOCKET, SO_RCVBUF, &size, sizeof(size));
        setsockopt(sockets[1], SOL_SOCKET, SO_SNDBUF, &size, sizeof(size));
        setsockopt(sockets[1], SOL_SOCKET, SO_RCVBUF, &size, sizeof(size));
        fcntl(sockets[0], F_SETFL, O_NONBLOCK);
        fcntl(sockets[1], F_SETFL, O_NONBLOCK);
        mReceiveFd = sockets[0];
        mSendFd = sockets[1];
    } else {
        mReceiveFd = -errno;
        ALOGE("BitTube: pipe creation failed (%s)", strerror(-mReceiveFd));
    }
}
```

- 其中：fds[0]就是对应的mReceiveFd,是管道的读端，sensor数据的读取端，对应的是客户端进程访问的。fds[1]就是对应mSendFd,是管道的写端,sensor数据写入端,是sensor的服务进程访问的一端。通过pipe(fds)创建管道,通过fcntl来设置操作管道的方式,设置通道两端的操作方式为O_NONBLOCK ，非阻塞IO方式，read或write调用返回-1和EAGAIN错误。总结下消息队列，客户端第一次注册监听器的时候，就需要创建一个消息队列，客户端创了SensorThread线程从消息队列里面读取数据。SensorEventQueue中有一个SensorEventConnection实例的引用，SensorEventConnection中有一个BitTube实例的引用。


### 使能Sensor

- 客户端创建了连接接口SensorEventConnection后，可以调用其方法使能Sensor传感器：

``` cpp
status_t SensorService::SensorEventConnection::enableDisable(
        int handle, bool enabled)
{
    status_t err;
    if (enabled) {
        err = mService->enable(this, handle);
    } else {
        err = mService->disable(this, handle);
    }
    return err;
}
```

- handle对应着Sensor传感器的句柄

### 服务端往管道写数据

``` cpp
bool SensorService::threadLoop()
{
    ……
    do {
        count = device.poll(buffer, numEventMax);

        recordLastValue(buffer, count);
        ……

        // send our events to clients...
        const SortedVector< wp<SensorEventConnection> > activeConnections(
                getActiveConnections());
        size_t numConnections = activeConnections.size();
        for (size_t i=0 ; i<numConnections ; i++) {
            sp<SensorEventConnection> connection(
                    activeConnections[i].promote());
            if (connection != 0) {
                connection->sendEvents(buffer, count, scratch);
            }
        }
    } while (count >= 0 || Thread::exitPending());
    return false;
}
```

- 前面介绍过，在SensorService中，创建了一个线程不断从HAL层读取Sensor数据，就是在threadLoop方法中。

- 关键在与下面了一个for循环，其实是扫描有多少个客户端连接接口，然后就往没每个连接的管道中写数据

``` cpp
status_t SensorService::SensorEventConnection::sendEvents(
        sensors_event_t const* buffer, size_t numEvents,
        sensors_event_t* scratch)
{
    // filter out events not for this connection
    size_t count = 0;
    if (scratch) {
      ……
    }
    ……
    if (count == 0)
        return 0;

    ssize_t size = mChannel->write(scratch, count*sizeof(sensors_event_t));
    ……
}
```

- 调用该连接接口的BitTube::write()，到此，服务端就完成了往管道的写端写入数据:

``` cpp
ssize_t BitTube::write(void const* vaddr, size_t size)
{
    ssize_t err, len;
    do {
        len = ::send(mSendFd, vaddr, size, MSG_DONTWAIT | MSG_NOSIGNAL);
        err = len < 0 ? errno : 0;
    } while (err == EINTR);
    return err == 0 ? len : -err;

}
```

### 客户端读管道数据

- 时序图:

<img src="https://raw.githubusercontent.com/way1989/way1989.github.io/master/uploads/images/sensorFramework/10.png"/>

``` cpp
ssize_t SensorEventQueue::read(ASensorEvent* events, size_t numEvents)
{
    return BitTube::recvObjects(mSensorChannel, events, numEvents);
}
```

- 调用到了BitTube::read()：

``` cpp
static ssize_t recvObjects(const sp<BitTube>& tube,
            T* events, size_t count) {
        return recvObjects(tube, events, count, sizeof(T));
    }
ssize_t BitTube::recvObjects(const sp<BitTube>& tube,
        void* events, size_t count, size_t objSize)
{
    ssize_t numObjects = 0;
    for (size_t i=0 ; i<count ; i++) {
        char* vaddr = reinterpret_cast<char*>(events) + objSize * i;
        ssize_t size = tube->read(vaddr, objSize);
        if (size < 0) {
            // error occurred
            return size;
        } else if (size == 0) {
            // no more messages
            break;
        }
        numObjects++;
    }
    return numObjects;
}

ssize_t BitTube::read(void* vaddr, size_t size)
{
    ssize_t err, len;
    do {
        len = ::recv(mReceiveFd, vaddr, size, MSG_DONTWAIT);
        err = len < 0 ? errno : 0;
    } while (err == EINTR);
    if (err == EAGAIN || err == EWOULDBLOCK) {
        return 0;
    }
    return err == 0 ? len : -err;
}
```
