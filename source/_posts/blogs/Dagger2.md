---
date: 2017-09-10 11:16:24
title: 解耦神器dagger2
description: dagger2入门以及深入理解
categories:   # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
 - dagger2
tags: # 这里写的标签会自动汇集到 tags 页面上
 - dagger2
---

## 一. Dagger2介绍

### 1.Dagger2是什么？

&emsp;&emsp;Dagger2是由Google接手开发，最早的版本Dagger1 是由Square公司开发的。大神[JakeWharton](https://github.com/JakeWharton)最近也从 Square 公司跳槽到 Google 💪💪。
```
A fast dependency injector for Android and Java
Android和Java的依赖快速注入器
```

### 2. Dagger2相较于Dagger1的优势是什么？

* **更好的性能**：相较于Dagger1，它使用的预编译期间生成代码来完成依赖注入，而不是用的反射。大家知道反射对手机应用开发影响是比较大的，因为反射是在程序运行时加载类来进行处理所以会比较耗时，而手机硬件资源有限，所以相对来说会对性能产生一定的影响。
* **容易跟踪调试**：因为dagger2是使用生成代码来实现完整依赖注入，所以完全可以在相关代码处下断点进行运行调试。

### 3. 使用依赖注入的最大好处是什么？

&emsp;&emsp;快速自动的构建出我们所需要的依赖对象，这里的依赖对象可以理解为某一个成员变量。例如在 `MVP` 中，`VP` 层就是互相关联的， `V` 要依赖对应的 `P`，而 `P` 也要依赖对应的 `V` 。dagger2 能解决的就是这种依赖关系，通过注入的方式，将双方的耦合再次降低，在实际的使用中体现为一个注解想要的对象就创建好了，咱们不用再去管理所依赖对象的创建等情况了。

### 4. 举个例子🌰

&emsp;&emsp;如果在 `MainActivity` 中，有 `Tinno`的实例，则称 `MainActivity` 对 `Tinno` 有一个依赖。如果不用Dagger2的情况下我们应该这么写：
```java
Tinno mTinno;

public MainActivity() {
    mTinno = new Tinno();
}
```
&emsp;&emsp;上面例子面临着一个问题，一旦某一天`Tinno`的创建方式（如构造参数）发生改变，那么你不但需要修改`MainActivity`中创建`Tinno`的代码，还要修改其他所有地方创建`Tinno`的代码。如果我们使用了Dagger2的话，就不需要管这些了，只需要在需要B的地方写下：
```java
@Inject
Tinno mTinno;
```

## 二. Dagger2使用

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

### 3. 结构

&emsp;&emsp;Dagger2要实现一个完整的依赖注入，必不可少的元素有三种：Module，Component，Container。为了便于理解，其实可以把component想象成针管，module是注射瓶，里面的依赖对象是注入的药水，build方法是插进患者（Container），inject方法的调用是推动活塞。
<div align="center"><img src="https://raw.githubusercontent.com/way1989/way1989.github.io/hexo/images_post/dagger2/1.png"/></div>

### 4. 简单的例子🌰

#### 4.1 需要依赖的对象

&emsp;&emsp;使用了注解方式，还是以`Tinno`为例，使得Dagger2能找到它。
```Java
public class Tinno {
    //这里可以看到加入了注解方式
    @Inject
    public Tinno() {

    }
}
```

#### 4.2 申明`Component`接口

&emsp;&emsp;申明完后rebuild一下工程，使其自动生成`Component`实现类`DaggerMainActivityComponent`。
```Java
//用这个标注标识是一个连接器
@Component
public interface MainActivityComponent {
    //这个连接器要注入的对象。这个inject标注的意思是，我后面的参数对象里面有标注为@Inject的属性，这个标注的属性是需要这个连接器注入进来的。
    void inject(MainActivity activity);
}
```

#### 4.3 在使用的地方注入

```Java
public class MainActivity extends AppCompatActivity {
    private static final String TAG = "MainActivity";
    @Inject
    Tinno mTinno;//加入注解，标注这个Tinno是需要注入的

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        TextView dagger2TextView = (TextView) findViewById(R.id.dagger2_text_view);
        //使用组件进行构造，注入
        DaggerMainActivityComponent.builder().build().inject(this);
        Log.d(TAG, "onCreate: mTinno = " + mTinno);

        dagger2TextView.setText(mTinno.toString());
    }
}
```

#### 4.4 小结

&emsp;&emsp;这是最简单的一种使用了。首先我们看到，第一印象是我去😲，这个更复杂了啊😂😂。我只能说确实，因为这个是它对的最基础的使用，看起来很笨拙，但是当它在大型项目里面，在依赖更多的情况下，则会发生质的飞跃，会发现它非常好用，并且将你需要传递的参数都隐藏掉，来实现解耦。
&emsp;&emsp;**Dagger2的注释思路**：关键的点是@Component，这个是个连接器，用来连接提供方和使用方的，所以它是桥梁。它使用在组件里面标记使用的Module（标记用到了哪个Module，主要是看使用方需要哪些对象进行构造，然后将它的提供方@module写在这里） 然后我们写入一个void inject(MainActivity activity); 这里后面的参数，就是我们的使用方了。如此一来，我们在使用的地方，使用类似这种方式`DaggerMainActivityComponent.builder().build().inject(this);`的动作，将使用方类里面的标记 为@Inject的类初始化掉，完成自动初始化的动作。
