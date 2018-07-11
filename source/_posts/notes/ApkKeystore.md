---
date: 2016-08-04 11:16:24
title: 将.pem与.pk8文件转换成.keystore签名文件
description: 简单的几个命令将.pem与.pk8文件转换成.keystore签名文件
categories:   # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
 - 笔记
 - keystore
tags: # 这里写的标签会自动汇集到 tags 页面上
 - 笔记
---

## 新建一个platform目录，将平台用到的两个文件platform.x509.pem和platform.pk8拷贝过来，通常在build/target/product/security/目录下,普通签名方式是：
``` bash
#!/bin/bash
inPath=$1
outPath=$2

java -jar out/host/linux-x86/framework/signapk.jar build/target/product/security/tinno_common/platform.x509.pem  build/target/product/security/tinno_common/platform.pk8 ${inPath} ${outPath}
```

## 把pkcs8格式的私钥转换为pkcs12格式，生成platform.priv.pem文件：
``` bash
$ openssl pkcs8 -in platform.pk8 -inform DER -outform PEM -out platform.priv.pem -nocrypt
```

## 生成pkcs12格式的密钥文件,生成platform.pk12文件，最后的brilliance是keystore的alias，需要输入两次密码，我们这里默认为android。
``` bash
$ openssl pkcs12 -export -in platform.x509.pem -inkey platform.priv.pem -out platform.pk12 -name brilliance
```

## 生成platform.keystore
``` bash
$ keytool -importkeystore -deststorepass android -destkeypass android -destkeystore platform.keystore -srckeystore platform.pk12 -srcstoretype PKCS12 -srcstorepass android -alias brilliance
```

## 使用.keystore签名的脚本signapk：
``` bash
#!/bin/bash
# Sample usage is as follows;
# ./signapk myapp.apk debug.keystore android androiddebugkey
#
# param1, APK file: Calculator_debug.apk
# param2, keystore location: ~/.android/debug.keystore
# param3, key storepass: android
# param4, key alias: androiddebugkey

USER_HOME=$(eval echo ~${SUDO_USER})

# use my debug key default
APK=$1
KEYSTORE="${2:-$USER_HOME/.android/debug.keystore}"
STOREPASS="${3:-android}"
ALIAS="${4:-androiddebugkey}"


# get the filename
APK_BASENAME=$(basename $APK)
SIGNED_APK="signed_"$APK_BASENAME

#debug
echo param1 $APK
echo param2 $KEYSTORE
echo param3 $STOREPASS
echo param4 $ALIAS

# delete META-INF folder
zip -d $APK META-INF/\*

# sign APK
jarsigner -verbose -sigalg SHA1withRSA -digestalg SHA1 -keystore $KEYSTORE -storepass $STOREPASS $APK $ALIAS
#verify
jarsigner -verify $APK

#zipalign
zipalign -v 4 $APK $SIGNED_APK
```
