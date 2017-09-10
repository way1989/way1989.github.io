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

&emsp;&emsp;Dagger2要实现一个完整的依赖注入，必不可少的元素有三种：__Module__，__Component__，__Container__。为了便于理解，其实可以把component想象成针管，module是注射瓶，里面的依赖对象是注入的药水，build方法是插进患者（Container），inject方法的调用是推动活塞。
<div align="center"><img src="https://raw.githubusercontent.com/way1989/way1989.github.io/hexo/images_post/dagger2/1.png"/></div>

### 4. 简单的例子🌰

&emsp;&emsp;__需要依赖的对象__，使用了注解方式，还是以`Tinno`为例，使得Dagger2能找到它。
```Java
public class Tinno {
    //这里可以看到加入了注解方式
    @Inject
    public Tinno() {

    }
}
```

&emsp;&emsp; __申明`Component`接口__，申明完后rebuild一下工程，使其自动生成`Component`实现类`DaggerMainActivityComponent`。
```Java
//用这个标注标识是一个连接器
@Component
public interface MainActivityComponent {
    //这个连接器要注入的对象。这个inject标注的意思是，我后面的参数对象里面有标注为@Inject的属性，这个标注的属性是需要这个连接器注入进来的。
    void inject(MainActivity activity);
}
```

&emsp;&emsp; __在使用的地方注入__

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

&emsp;&emsp; __简单使用小结__
&emsp;&emsp; 这是最简单的一种使用了。首先我们看到，第一印象是我去😲，这个更复杂了啊😂😂。我只能说确实，因为这个是它对的最基础的使用，看起来很笨拙，但是当它在大型项目里面，在依赖更多的情况下，则会发生质的飞跃，会发现它非常好用，并且将你需要传递的参数都隐藏掉，来实现解耦。

### 5. 常规使用方法
&emsp;&emsp;细心的朋友发现了，我上一节讲 dagger2结构的时候提到必不可少的三元素，这个例子只用到了 __Component__，__Container__，__Module__ 并未提及，这就是我这里需要重点提及的，通过以下这个例子，能更加深刻的理解 __Module__ 的作用。
&emsp;&emsp;实现一个 MainModule，提供一些实例构造，通过 __Component__ 联系起来。
```Java
@Module //实现一个类，标注为 Module
public class MainModule {

    @Provides //实现一些提供方法，供外部使用
    public Tinno provideTinno(){
        return new Tinno();
    }
}
```

&emsp;&emsp;在 MainComponent 中，指明Component查找Module的位置
```Java
@Component(modules = MainModule.class)
public interface MainActivityComponent {// 必须定义为接口，Dagger2框架将自动生成Component的实现类，对应的类名是Dagger×××××，这里对应的实现类是DaggerMainActivityComponent
    void inject(MainActivity activity);// 注入到MainActivity(Container)的方法，方法名一般使用inject
}
```
&emsp;&emsp;最后我们的`Tinno`类中的`@Inject`和构造函数可以去掉了(亲测不去掉也是可以正常运行的，此时也是使用`Module`中提供的对象，具体可以通过后面分享的`@Scope`来验证)，如下所示。
```Java
public class Tinno {
}
```
&emsp;&emsp;注入使用的地方完全不用修改，也能得到和之前例子一样的结果。

&emsp;&emsp;**Dagger2的注释思路**：关键的点是@Component，这个是个连接器，用来连接提供方和使用方的，所以它是桥梁。它使用在组件里面标记使用的Module（标记用到了哪个Module，主要是看使用方需要哪些对象进行构造，然后将它的提供方@module写在这里） 然后我们写入一个void inject(MainActivity activity); 这里后面的参数，就是我们的使用方了。如此一来，我们在使用的地方，使用类似这种方式`DaggerMainActivityComponent.builder().build().inject(this);`的动作，将使用方类里面的标记 为@Inject的类初始化掉，完成自动初始化的动作。

### 6. 更多用法
#### 6.1 方法参数
&emsp;&emsp;上面的例子@Provides标注的方法是没有输入参数的，Module中@Provides标注的方法是可以带输入参数的，其参数值可以由Module中的其他被@Provides标注的方法提供。
```Java
@Module //实现一个类，标注为 Module
public class MainModule {
    private Context mContext;

    public MainModule(Context context) {
        mContext = context;
    }

    @Provides //实现一些提供方法，供外部使用
    public Tinno provideTinno(Gson gson, CameraTeam cameraTeam) {
        return new Tinno(mContext, gson, cameraTeam);
    }

    @Provides
    public Gson provideGson() {
        return new GsonBuilder()
                .excludeFieldsWithModifiers(Modifier.PROTECTED)//忽略protected字段
                .setDateFormat("yyyy-MM-dd'T'HH:mm:ssZ")
                .create();

    }

//    @Provides
//    public CameraTeam provideCameraTeam() {
//        return new CameraTeam();
//    }
}
```
&emsp;&emsp;如果找不到被@Provides注释的方法提供对应参数对象的话，将会自动调用被@Inject注释的构造方法生成相应对象。
```Java
public class CameraTeam {
    @Inject
    public CameraTeam() {

    }
}
```
&emsp;&emsp;由于我们修改了`MainModule`，所以对应注入的地方要稍微修改一下：
```Java
//注意此处比之前多了.mainModule(new MainModule(getApplicationContext()))
DaggerMainActivityComponent.builder().mainModule(new MainModule(getApplicationContext())).build().inject(this);
```

__思考__
&emsp;&emsp; 通过上面3个例子`@Provides`和`@Inject`两种方式提供对象的区别？

#### 6.2 添加多个Module

&emsp;&emsp;一个Component可以添加多个Module，这样Component获取依赖时候会自动从多个Module中查找获取。添加多个Module有两种方法，一种是在Component的注解@Component(modules={××××，×××})中添加多个modules。
```Java
@Component(modules={MainModule.class,ModuleA.class,ModuleB.class,ModuleC.class}) //直接在Component引用多个 Module
public interface MainActivityComponent {
    ...
}
```
&emsp;&emsp;另外一种添加多个Module的方法可以使用@Module的 includes的方法（includes={××××，×××}）。
```Java
@Module(includes={ModuleA.class,ModuleB.class,ModuleC.class})//先在一个 Module 中includes其他 Module
public class MainModule {
    ...
}
@Component(modules={MainModule.class}) //只有一个 Module 时可以不用{}
public interface MainActivityComponent {
    ...
}
```

#### 6.3 区分返回类型相同@Provides方法

&emsp;&emsp;如果我们在 `Module` 中有重复的类型返回，例如我定义两个 Context 类型的provides 在 Module 中的话，编译直接会报错：

```
Error:(16, 10) 错误: android.content.Context is bound multiple times:
@Provides android.content.Context com.ape.dagger2.MainModule.provideApplicationContext()
@Provides android.content.Context com.ape.dagger2.MainModule.provideActivityContext()
```
&emsp;&emsp;那如果我们真的需要注入同一类型多次呢，这个问题总会有解决方案的吧？要是真的这么坑估计也没人用 dagger 了吧！哈哈。。。😂  其实 dagger2 为我们提供了两种方式来解决这个问题：__可以使用`@Qualifier` 的注解来区分，或者其子类 `@Named("xx")` 的注解__。

__@Named 方式__
```Java
@Module //实现一个类，标注为 Module
public class MainModule {
    private Context mApplicationContext;
    private Context mActivityContext;

    public MainModule(Context context, Context activityContext) {
        mApplicationContext = context;
        mActivityContext = activityContext;
    }

    @Provides //实现一些提供方法，供外部使用
    public Tinno provideTinno(@Named("application")/*使用的是 application*/Context context, Gson gson, CameraTeam cameraTeam) {
        return new Tinno(context, gson, cameraTeam);
    }

    @Named("application") //标注为 application
    @Provides
    public Context provideApplicationContext() {
        return mApplicationContext;
    }

    @Named("activity") //标注为 activity
    @Provides
    public Context provideActivityContext() {
        return mActivityContext;
    }
  ...
}
```

__@Qualifier方式__
```Java
@Qualifier
@Documented
@Retention(RUNTIME)
public @interface ApplicationQualifier {
}
```
```Java
@Qualifier
@Documented
@Retention(RUNTIME)
public @interface ActivityQualifier {
}
```
```java
@Module //实现一个类，标注为 Module
public class MainModule {
    private Context mApplicationContext;
    private Context mActivityContext;

    public MainModule(Context context, Context activityContext) {
        mApplicationContext = context;
        mActivityContext = activityContext;
    }

    @Provides //实现一些提供方法，供外部使用
    public Tinno provideTinno(@ApplicationQualifier/*此处使用为 ApplicationQualifier*/ Context context, Gson gson, CameraTeam cameraTeam) {
        return new Tinno(context, gson, cameraTeam);
    }

    @ApplicationQualifier //标注为 ApplicationQualifier
    @Provides
    public Context provideApplicationContext() {
        return mApplicationContext;
    }

    @ActivityQualifier //标注为 ActivityQualifier
    @Provides
    public Context provideActivityContext() {
        return mActivityContext;
    }
    ...
}
```
&emsp;&emsp;使用哪种方式就仁者见仁智者见智了，但个人推荐使用`@Qualifier`，毕竟输入太多字符串容易出错。

#### 6.4 组件间依赖和子组件
&emsp;&emsp;有时我们需要依赖一个组件，这个最常见的用法是，如果我们定义了 MainActivity 的 `MainComponent` ，并且它依赖咱们的 `AppComponent` 里面的 `IRepositoryManager` 的话就要这样定义了：
```Java
@Component(dependencies = AppComponent.class, modules = MainPresenterModule.class)
public interface MainComponent {
    void inject(MainActivity activity);
}
```
&emsp;&emsp;在 `AppComponent` 中需要将获取 `IRepositoryManager` 的方法暴露出来，不然还是无法注入成功的。
```Java

@Component(modules = {AppModule.class})
public interface AppComponent {
    //用于管理网络请求层,以及数据缓存层,对外开放的接口
    IRepositoryManager repositoryManager();
}
```
&emsp;&emsp;那如果我觉得暴露这些方法太麻烦了，那需要怎么办呢？最简单就是使用 `@SubComponent` ,在所属的父 `Component` 中定义一个 `SubComponent`，该 `SubComponent` 中将会包含父 `Component` 的所有方法。父 `Component` 不显示声明都可以。
```Java
@Subcomponent(modules = MainPresenterModule.class)
public interface MainComponent {
    void inject(MainActivity activity);
}
```
```Java
@Component(modules = {AppModule.class})
public interface AppComponent {
    // 提供 MainComponent 对象的获取方法
    MainComponent mainComponent(MainPresenterModule module);
}
```
&emsp;&emsp;如何确保我们的MainComponent就是AppComponent的子组件呢？直接在注入的时候指明：
```Java
appComponent.mainComponent(new MainPresenterModule(this)).inject(this);
//DaggerMainComponent.builder().appComponent(appComponent)
//                .mainPresenterModule(new MainPresenterModule(this)).build().inject(this);
```

&emsp;&emsp; __组件依赖和子组件的区别__：
| 组件依赖 | 子组件                          |
|---------|----------------------------------|
| 1. 保持两个 Component 都独立，没有任何关联<br />2. 明确的告诉别人这个 Component 所依赖的 Component <br />3. 两个拥有依赖关系的 Component 是不能有相同 @Scope 注解的  | 1. 保持两个 Component 内聚<br />2. 不关心这个 Component 依赖哪个 Component<br />3. 可以使用相同的@Scope注解|


#### 6.5 懒加载和强加载模式
&emsp;&emsp;在上面的比喻中，一针扎进去，是撒都给你打进去了，那么如果有些我想要在调用的时候才加载呢？这里 dagger 提供了 Lazy<T> 的方式来注入，对应的获取就是：
```Java
public class Container{
    @Inject Lazy<Tinno> mTinnoLazy; //延迟加载
    @Inject Provider<Tinno> mTinnoProvider;//实现强制加载，每次调用get都会调用Module的Provides方法一次，和懒加载模式正好相反,比如我们需要一次性创建出10个Tinno 对象
    public void init(){
        DaggerComponent.create().inject(this);
        Tinno tinno = mTinnoLazy.get();  //调用get时才创建b
    }
}
```

#### 6.6 `@Scope`详解
__@Scope 是什么__
&emsp;&emsp; scope 翻译过来就是辖域，再结合到计算机上，其实就是作用域的意思，学过高级语言的应该都知道设计模式中一个模式叫做单例模式，单例即为全局中该对象的实例只存在一个，而在 dagger2 中，@scope 的一个默认实现就是 @Singleton，乍一看，很神奇啊，仅仅使用一个注解就可以实现单例！

__@Scpoe 怎么用__
&emsp;&emsp; 那么接下来我们就看一下它的使用。代码如下：
```Java
public class User {}

@Module
public class UserModule {
    @Provides
    @Singleton
    User provideUser() {
        return new User();
    }
}

@Singleton
@Component(modules = UserModule.class)
public interface UserComponent {
    void inject(MainActivity activity);
    void inject(SecondActivity activity);
}
```
&emsp;&emsp; 我们创建一个普通的 User 类，然后创建它的 Module，并且用 @Singleton 标记该 User 返回对象，最后我们再创建它的 Component，然后用 @Singleton 标记这个 Component。这是一个标准的套路流程。接下来我们创建一个 MainActivity 和一个 SecondActivity，代码如下：
```Java
public class MainActivity extends AppCompatActivity {
    @Inject
    User mUser1;
    @Inject
    User mUser2;
    private TextView mContentTextView;
    private Button mContentButton;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mContentTextView = (TextView) findViewById(R.id.tv_content);
        mContentButton = (Button) findViewById(R.id.btn_content);

        // [1]
        UserComponent component = DaggerUserComponent.create();
        component.inject(this);
        // 第一行为 mUser1 的信息，第二行为 mUser2 的信息，第三行为该类中 UserComponent 的信息
        mContentTextView.setText(mUser1.toString() + "\n" + mUser2.toString()+"\n"+ component.toString());
        mContentButton.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                startActivity(new Intent(MainActivity.this, SecondActivity.class));
            }
        });
    }
}

public class SecondActivity extends AppCompatActivity {
    @Inject
    User mUser;
    private TextView mContentTextView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_second);
        mContentTextView = (TextView) findViewById(R.id.tv_content);
        // [2]
        UserComponent component = DaggerUserComponent.create();
        component.inject(this);
        // 第一行为 mUser 的信息，第二行为该类中 UserComponent 的信息
        mContentTextView.setText(mUser.toString() + "\n" + component.toString());
    }
}
```
