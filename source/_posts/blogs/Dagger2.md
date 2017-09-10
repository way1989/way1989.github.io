---
date: 2017-09-10 11:16:24
title: 深入浅出dagger2
description: dagger2入门以及深入理解
categories:   # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
 - dagger2
tags: # 这里写的标签会自动汇集到 tags 页面上
 - dagger2
---

## Dagger2介绍

### 1.Dagger2是什么？

&emsp;&emsp;Dagger2是由Google接手开发，最早的版本Dagger1 是由Square公司开发的。大神[JakeWharton](https://github.com/JakeWharton)最近也从 Square 公司跳槽到 Google😂😂。
```
A fast dependency injector for Android and Java
Android和Java的依赖快速注入器
```

### 2. Dagger2相较于Dagger1的优势是什么？

* **更好的性能**：相较于Dagger1，它使用的预编译期间生成代码来完成依赖注入，而不是用的反射。大家知道反射对手机应用开发影响是比较大的，因为反射是在程序运行时加载类来进行处理所以会比较耗时，而手机硬件资源有限，所以相对来说会对性能产生一定的影响。
* **容易跟踪调试**：因为dagger2是使用生成代码来实现完整依赖注入，所以完全可以在相关代码处下断点进行运行调试。

### 3. 使用依赖注入的最大好处是什么？

&emsp;&emsp;快速自动的构建出我们所需要的依赖对象，这里的依赖对象可以理解为某一个成员变量。例如在 `MVP` 中，`VP` 层就是互相关联的， `V` 要依赖对应的 `P`，而 `P` 也要依赖对应的 `V` 。dagger2 能解决的就是这种依赖关系，通过注入的方式，将双方的耦合再次降低，在实际的使用中体现为一个注解想要的对象就创建好了，咱们不用再去管理所依赖对象的创建等情况了。

### 4. 举个例子！

&emsp;&emsp;如果在 Class A 中，有 Class B 的实例，则称 Class A 对 Class B 有一个依赖。如果不用Dagger2的情况下我们应该这么写：
```java
...
B b;
...
public A() {
    b = new B();
}
}
```
&emsp;&emsp;上面例子面临着一个问题，一旦某一天B的创建方式（如构造参数）发生改变，那么你不但需要修改A中创建B的代码，还要修改其他所有地方创建B的代码。如果我们使用了Dagger2的话，就不需要管这些了，只需要在需要B的地方写下：
```java
@Inject
B b;
```

## Dagger2使用

### 1. gradle配置

&emsp;&emsp;Android Studio 2.2以前的版本需要使用Gradle插件`android-apt`(Annotation Processing Tool)，协助Android Studio处理`annotation processors`；`annotationProcessor`就是APT工具中的一种，他是google开发的内置框架，不需要引入，所以可以像下面这样直接使用。
```Groovy
// Add Dagger dependencies
dependencies {
  compile 'com.google.dagger:dagger:2.4'
  annotationProcessor 'com.google.dagger:dagger-compiler:2.4'
}
```

### 2. 注解

&emsp;&emsp;Dagger2 通过注解来生成代码，定义不同的角色，主要的注解如下：
* **@Module**: Module类里面的方法专门提供依赖，所以我们定义一个类，用@Module注解，这样Dagger在构造类的实例的时候，就知道从哪里去找到需要的依赖。
* **@Provides**: 在Module中，我们定义的方法是用这个注解，以此来告诉Dagger我们想要构造对象并提供这些依赖。
* **@Inject**: 通常在需要依赖的地方使用这个注解。换句话说，你用它告诉Dagger这个类或者字段需要依赖注入。这样，Dagger就会构造一个这个类的实例并满足他们的依赖。
* **@Component**: Component从根本上来说就是一个注入器，也可以说是@Inject和@Module的桥梁，它的主要作用就是连接这两个部分。将Module中产生的依赖对象自动注入到需要依赖实例的Container中。
* **@Scope**: Dagger2可以通过自定义注解限定注解作用域，来管理每个对象实例的生命周期。
* **@Qualifier**: 当类的类型不足以鉴别一个依赖的时候，我们就可以使用这个注解标示。例如：在Android中，我们会需要不同类型的context，所以我们就可以定义 qualifier注解“@perApp”和“@perActivity”，这样当注入一个context的时候，我们就可以告诉 Dagger我们想要哪种类型的context。
