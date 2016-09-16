---
date: 2016-09-14 15:24:41
title: Android Sensor框架HAL层解读
description: Android Sensor框架HAL层浅度解析
categories:   # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
 - 博客
 - Sensor
tags: # 这里写的标签会自动汇集到 tags 页面上
 - 博客
 - Sensor
---

## Android sensor构建

1. Android6.0 系统内置对传感器的支持达26种，他们分别是：加速度传感器(accelerometer)、磁力传感器(magnetic field)、方向传感器(orientation)、陀螺仪(gyroscope)、环境光照传感器(light)、压力传感器(pressure)、温度传感器(temperature)和距离传感器(proximity)等。

2. Android实现传感器系统包括以下几个部分：
    - java层
    - JNI层
    - HAL层
    - 驱动层

3. 各部分之间架构图如下：

<img src="https://raw.githubusercontent.com/way1989/way1989.github.io/master/uploads/images/sensorHAL/1.png"/>


## Sensor HAL层接口

1. Google为Sensor提供了统一的HAL接口，不同的硬件厂商需要根据该接口来实现并完成具体的硬件抽象层。Android中Sensor的HAL接口定义在：`hardware/libhardware/include/hardware/sensors.h`

2. 对传感器类型的定义:
```cpp
#define SENSOR_TYPE_ACCELEROMETER                   (1)  //加速度传感器
#define SENSOR_TYPE_MAGNETIC_FIELD                  (2)  //磁力传感器
#define SENSOR_TYPE_ORIENTATION                     (3)  //方向
#define SENSOR_TYPE_GYROSCOPE                       (4)  //陀螺仪
#define SENSOR_TYPE_LIGHT                           (5)  //环境光照传感器
#define SENSOR_TYPE_PRESSURE                        (6)  //压力传感器
#define SENSOR_TYPE_TEMPERATURE                     (7)  //温度传感器
#define SENSOR_TYPE_PROXIMITY                       (8)  //距离传感器
#define SENSOR_TYPE_GRAVITY                         (9)  //重力
#define SENSOR_TYPE_LINEAR_ACCELERATION             (10) //线性加速度
#define SENSOR_TYPE_ROTATION_VECTOR                 (11) //运动
#define SENSOR_TYPE_RELATIVE_HUMIDITY               (12) //湿度传感器
#define SENSOR_TYPE_AMBIENT_TEMPERATURE             (13) //环境温度传感器
#define SENSOR_TYPE_MAGNETIC_FIELD_UNCALIBRATED     (14) //未校准磁力传感器
#define SENSOR_TYPE_GAME_ROTATION_VECTOR            (15) //游戏动作传感器
#define SENSOR_TYPE_GYROSCOPE_UNCALIBRATED          (16) //未校准陀螺仪
#define SENSOR_TYPE_SIGNIFICANT_MOTION              (17) //特殊动作触发传感器
#define SENSOR_TYPE_STEP_DETECTOR                   (18) //步行检测传感器
#define SENSOR_TYPE_STEP_COUNTER                    (19) //计步传感器
#define SENSOR_TYPE_GEOMAGNETIC_ROTATION_VECTOR     (20) //地磁旋转矢量传感器,提供手机的旋转矢量
#define SENSOR_TYPE_HEART_RATE                      (21) //心率传感器
#define SENSOR_TYPE_TILT_DETECTOR                   (22) //每次检测到倾斜事件后均生成事件
#define SENSOR_TYPE_WAKE_GESTURE                    (23) //支持根据设备特定的动作唤醒设备
#define SENSOR_TYPE_GLANCE_GESTURE                  (24) //支持短暂打开屏幕，以便用户根据特定动作浏览屏幕上的内容
#define SENSOR_TYPE_PICK_UP_GESTURE                 (25) //拾起设备时触发,无论面前是什么(桌子、口袋、手提袋)
#define SENSOR_TYPE_WRIST_TILT_GESTURE              (26) //腕关节倾斜触发生成事件
```

3. 传感器模块的定义结构体如下，该接口的定义实际上是对标准的硬件模块hw_module_t的一个扩展，增加了一个`get_sensors_list`函数，用于获取传感器的列表。
```cpp
struct sensors_module_t {
    struct hw_module_t common;
    int (*get_sensors_list)(struct sensors_module_t* module,
            struct sensor_t const** list);
};
```

4. 对任意一个sensor设备都会有一个`sensor_t`结构体，其定义如下：
```cpp
struct sensor_t {
    const char*     name;       //传感器名字
    const char*     vendor;
    int             version;     //版本
    int             handle;     //传感器的handle句柄
    int             type;       //传感器类型
    float           maxRange;   //最大范围
    float           resolution;    //解析度
    float           power;       //消耗能源
    int32_t         minDelay;    //事件间隔最小时间
    void*           reserved[8];   //保留字段，必须为0
};
```

5. 每个传感器的数据由`sensors_event_t`结构体表示，定义如下，其中，sensor为传感器的标志符，而不同的传感器则采用union方式来表示。
```cpp
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

6. `sensors_vec_t`结构体用来表示不同传感器的数据：
```cpp
typedef struct {
    union {
        float v[3];
        struct {
            float x;
            float y;
            float z;
        };
        struct {
            float azimuth;
            float pitch;
            float roll;
        };
    };
    int8_t status;
    uint8_t reserved[3];
} sensors_vec_t;
```

7. Sensor设备结构体`sensors_poll_device_t`，对标准硬件设备`hw_device_t`结构体的扩展，主要完成读取底层数据，并将数据存储在`struct sensors_poll_device_t`结构体中；poll函数用来获取底层数据，调用时将被阻塞定义如下：
```cpp
struct sensors_poll_device_t {
struct hw_device_t common;
//Activate/deactivate one sensor
    int (*activate)(struct sensors_poll_device_t *dev,
            int handle, int enabled);
    //Set the delay between sensor events in nanoseconds for a given sensor.
    int (*setDelay)(struct sensors_poll_device_t *dev,
            int handle, int64_t ns);
    //获取数据
    int (*poll)(struct sensors_poll_device_t *dev,
            sensors_event_t* data, int count);
};
```

8. 控制设备打开/关闭结构体定义如下：
```cpp
static inline int sensors_open(const struct hw_module_t* module,
        struct sensors_poll_device_t** device) {
    return module->methods->open(module,
            SENSORS_HARDWARE_POLL, (struct hw_device_t**)device);
}

static inline int sensors_close(struct sensors_poll_device_t* device) {
    return device->common.close(&device->common);
}
```

## Sensor HAL实现

1. 打开设备时序图

<img src="https://raw.githubusercontent.com/way1989/way1989.github.io/master/uploads/images/sensorHAL/2.jpg"/>

2. SensorDevice属于JNI层，与HAL进行通信的接口，在JNI层调用了HAL层的`open_sensors()`方法打开设备模块，再调用`poll__activate()`对设备使能，然后调用`poll__poll`读取数据。
