---
date: 2017-01-16 23:02:24
title: Android运行时资源覆盖
description: Android Runtime Resource Overlay
categories:   # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
 - 博客
tags: # 这里写的标签会自动汇集到 tags 页面上
 - 博客
---

## 什么是运行时资源覆盖

无需修改原app的代码，无需重新编译，就能覆盖app内的部分资源，从而实现修改app的部分UI,后面简称RRO。

## RRO原理

* 应用通过 getString/getDrawable去调用某个资源时,会将这个resources ID 作为参数传给 framework 层。同一名称但不同状态的 resources ID 是一样的,比如不同分辨率但名称相同的图片分别被放置在了drawable-hdpi/drawable-ldpi/drawable-mdpi下，但在编译时针对该图片生成的resources ID只有一个。

* 为了快速查找到指定的资源，Apk编译的时候会把Java文件里面的R.String.app_name替换成ox7f123456这种格式的值

* 每个apk里面都有一个文件(resources.arsc)记录着指定的resource_id对应的资源类型，如果是string类型，则记录的这个资源名称对应的所有语言的翻译，如果是drawable类型，则记录着哪些分辨率底下有这个资源。

## Resources.arsc的结构 string

<img src="https://raw.githubusercontent.com/way1989/way1989.github.io/hexo/images_post/runtimeResourceOverlay/2017-01-16-23-12-24.jpg"/>

<img src="https://raw.githubusercontent.com/way1989/way1989.github.io/hexo/images_post/runtimeResourceOverlay/2017-01-16-23-12-40.jpg"/>

## Resources.arsc的结构 drawable

<img src="https://raw.githubusercontent.com/way1989/way1989.github.io/hexo/images_post/runtimeResourceOverlay/2017-01-16-23-13-30.jpg"/>

## RRO实现方法

* 建立一个新的overlay apk项目，apk项目底下有 res文件夹，Android.mk、AndroidManifest.xml
* 修改AndroidManifest.xml 配置apk的包名、要覆盖的apk的包名
* 修改Android.mk 使得apk能集成到system/vendor/overlay/目录下
* 找到需要覆盖的资源名，在overlay apk的res指定目录下定义同名的资源

## AndroidManifest.xml的写法

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"  
	package="com.android.gallery3d.overlay">  
	<overlay android:targetPackage="com.android.gallery3d" android:priority="1"/>  
</manifest> 
```

## Android.mk的写法及自动生成版本号的实现

```makefile
ifeq ($(strip $(MYOS_APE_GALLERY31_SUPPORT)), yes)
ifneq ($(strip $(MYOS_APE_GALLERY31_NAME)),)
$(warning --Android.mk--ApeGalleryOverlay31---------)

LOCAL_PATH:= $(call my-dir)

include $(CLEAR_VARS)

LOCAL_MODULE_TAGS := optional
LOCAL_SRC_FILES := $(call all-java-files-under, res)
LOCAL_PACKAGE_NAME := ApeGalleryOverlay31
LOCAL_CERTIFICATE := platform
LOCAL_MODULE_PATH := $(TARGET_OUT)/vendor/overlay

version1 = 6
version2 = 0
version3 = $(shell cd $(LOCAL_PATH) && git rev-list --count HEAD)
build_date = $(shell date +%y%m%d)
last_commit = $(shell cd $(LOCAL_PATH) && git describe --always)

version_code = $(shell expr $(version1) \* 1000000 + $(version2) \* 10000 + $(version3))
version_name := $(version1).$(version2).$(version3).$(build_date).$(last_commit)

$(warning --ApeGalleryOverlay31---$(version_code)---$(version_name)---)

LOCAL_AAPT_FLAGS += --version-code $(version_code)
LOCAL_AAPT_FLAGS += --version-name $(version_name)

include $(BUILD_PACKAGE)

endif
endif
```

## 自动生成版本号的意义

* 出现问题以后反查问题原因,是代码问题，还是集成问题
* 这里为什么要使用自动版本号:
    * 不用每次改版本号的麻烦
    * 版本号有序自增，没有出错的风险
    * 每一个编译出来的apk,都能定位到当时的最后一个提交
    * 由于是随系统编译,可能你没来得及改版本号就编译了，所以手工改版本号可能只能定位到大致的位置
* 注意事项:
    * 仓库提交9999次以后，版本会溢出，此时可以脚本向前进位

## 注意事项

* 不能覆盖layout、AndroidManifest.xml、asset目录
* 建议只覆盖字符资源、覆盖图片要谨慎
* Overlay apk中不要添加其它的东西，比方说style.xml 可能会导致被覆盖的apk打开就报错
* Overlay apk 必须要签名，否则不生效 
* 为了安全起见，Overylay的apk必须放到system/vendor/overlay/目录下才会生效
* 由于是把源码放在基线里面编译，所以提交代码需要特别小心，不要造成项目编译不过

## RRO对我们目前来说使用的意义

* 对于字符翻译需求，快速输出版本
* DCC以后，只有翻译需求，无需更新主apk版本