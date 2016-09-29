---
date: 2016-09-29 23:33:48
title: Android深入浅出 Retrofit
description: Android深入浅出 Retrofit
categories:   # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
 - 博客
 - Retrofit
tags: # 这里写的标签会自动汇集到 tags 页面上
 - 博客
 - Retrofit
---

<img src="https://raw.githubusercontent.com/way1989/way1989.github.io/hexo/source/postImage/Retrofit/1.jpg"/>

Android 开发中，从原生的 HttpUrlConnection 到经典的 Apache 的 HttpClient，再到对前面这些网络基础框架的封装，比如 Volley、Async Http Client，Http 相关开源框架的选择还是很多的，其中由著名的 Square 公司开源的 Retrofit 更是以其简易的接口配置、强大的扩展支持、优雅的代码结构受到大家的追捧。也正是由于 Square 家的框架一如既往的简洁优雅，所以我一直在想，Square 公司是不是只招处女座的程序员？

## 初识 Retrofit

单从 Retrofit 这个单词，你似乎看不出它究竟是干嘛的，当然，我也看不出来 :)逃。。
```
Retrofitting refers to the addition of new technology or features to older systems.
—From Wikipedia
```
于是我们就明白了，冠以 Retrofit 这个名字的这个家伙，应该是某某某的 『Plus』 版本了。

### Retrofit 概览

Retrofit 是一个 RESTful 的 HTTP 网络请求框架的封装。注意这里并没有说它是网络请求框架，主要原因在于网络请求的工作并不是 Retrofit 来完成的。Retrofit 2.0 开始内置 OkHttp，前者专注于接口的封装，后者专注于网络请求的高效，二者分工协作，宛如古人的『你耕地来我织布』，小日子别提多幸福了。
<img src="https://raw.githubusercontent.com/way1989/way1989.github.io/hexo/source/postImage/Retrofit/2.jpg"/>
我们的应用程序通过 Retrofit 请求网络，实际上是使用 Retrofit 接口层封装请求参数、Header、Url 等信息，之后由 OkHttp 完成后续的请求操作，在服务端返回数据之后，OkHttp 将原始的结果交给 Retrofit，后者根据用户的需求对结果进行解析的过程。
讲到这里，你就会发现所谓 Retrofit，其实就是 Retrofitting OkHttp 了。

### Hello Retrofit

多说无益，不要来段代码陶醉一下。使用 Retrofit 非常简单，首先你需要在你的 build.gradle 中添加依赖：
```groovy
compile 'com.squareup.retrofit2:retrofit:2.0.2'
```
你一定是想要访问 GitHub 的 api 对吧，那么我们就定义一个接口：
```java
public interface GitHubService {
  @GET("users/{user}/repos")
  Call<List<Repo>> listRepos(@Path("user") String user);
}
```
接口当中的 listRepos 方法，就是我们想要访问的 api 了：
`https://api.github.com/users/{user}/repos`
其中，在发起请求时， {user} 会被替换为方法的第一个参数 user。
好，现在接口有了，我们要构造 Retrofit 了：
```java
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("https://api.github.com/")
    .build();
 
GitHubService service = retrofit.create(GitHubService.class);
```
这里的 service 就好比我们的快递哥，还是往返的那种哈~
```java
Call<List<Repo>> repos = service.listRepos("octocat");
```
发请求的代码就像前面这一句，返回的 repos 其实并不是真正的数据结果，它更像一条指令，你可以在合适的时机去执行它：
```java
// 同步调用
List<Repo> data = repos.execute(); 
 
// 异步调用
repos.enqueue(new Callback<List<Repo>>() {
            @Override
            public void onResponse(Call<List<Repo>> call, Response<List<Repo>> response) {
                List<Repo> data = response.body();
            }
 
            @Override
            public void onFailure(Call<List<Repo>> call, Throwable t) {
                t.printStackTrace();
            }
        });
```
啥感觉？有没有突然觉得请求接口就好像访问自家的方法一样简单？呐，前面我们看到的，就是 Retrofit 官方的 demo 了。你以为这就够了？噗~怎么可能。。

### Url 配置

Retrofit 支持的协议包括 GET/POST/PUT/DELETE/HEAD/PATCH，当然你也可以直接用 HTTP 来自定义请求。这些协议均以注解的形式进行配置，比如我们已经见过 GET 的用法：
```java
@GET("users/{user}/repos")
Call<List<Repo>> listRepos(@Path("user") String user);
```
这些注解都有一个参数 value，用来配置其路径，比如示例中的 users/{user}/repos，我们还注意到在构造 Retrofit 之时我们还传入了一个 baseUrl("https://api.github.com/")，请求的完整 Url 就是通过 baseUrl 与注解的 value（下面称 “path“ ） 整合起来的，具体整合的规则如下：
* path 是绝对路径的形式：
     path = "/apath"，baseUrl = "http://host:port/a/b"
    Url = "http://host:port/apath"
* path 是相对路径，baseUrl 是目录形式：
    path = "apath"，baseUrl = "http://host:port/a/b/"
     Url = "http://host:port/a/b/apath"
* path 是相对路径，baseUrl 是文件形式：
     path = "apath"，baseUrl = "http://host:port/a/b"
     Url = "http://host:port/a/apath"
* path 是完整的 Url：
     path = "http://host:port/aa/apath"，baseUrl = "http://host:port/a/b"
     Url = "http://host:port/aa/apath"

建议采用第二种方式来配置，并尽量使用同一种路径形式。如果你在代码里面混合采用了多种配置形式，恰好赶上你哪天头晕眼花，信不信分分钟写一堆 bug 啊哈哈。

### 参数类型

发请求时，需要传入参数，Retrofit 通过注解的形式令 Http 请求的参数变得更加直接，而且类型安全。

#### Query & QueryMap

```java
@GET("/list")
Call<ResponseBody> list(@Query("page") int page);
```
Query 其实就是 Url 中 ‘?’ 后面的 key-value，比如：
`http://www.println.net/?cate=android`
这里的 cate=android 就是一个 Query，而我们在配置它的时候只需要在接口方法中增加一个参数，即可：
```java
interface PrintlnServer{
    @GET("/")
    Call<String> cate(@Query("cate") String cate);
}
```
这时候你肯定想，如果我有很多个 Query，这么一个个写岂不是很累？而且根据不同的情况，有些字段可能不传，这与方法的参数要求显然也不相符。于是，打群架版本的 QueryMap 横空出世了，使用方法很简单，我就不多说了。

#### Field & FieldMap

其实我们用 POST 的场景相对较多，绝大多数的服务端接口都需要做加密、鉴权和校验，GET 显然不能很好的满足这个需求。使用 POST 提交表单的场景就更是刚需了，怎么提呢？
```java
@FormUrlEncoded
@POST("/")
Call<ResponseBody> example(
    @Field("name") String name,
    @Field("occupation") String occupation);
```
其实也很简单，我们只需要定义上面的接口就可以了，我们用 Field 声明了表单的项，这样提交表单就跟普通的函数调用一样简单直接了。
等等，你说你的表单项不确定个数？还是说有很多项你懒得写？Field 同样有个打群架的版本——FieldMap，赶紧试试吧~~

#### Part & PartMap

这个是用来上传文件的。话说当年用 HttpClient 上传个文件老费劲了，一会儿编码不对，一会儿参数错误（也怪那时段位太低吧TT）。。。可是现在不同了，自从有了 Retrofit，妈妈再也不用担心文件上传费劲了~~~
```java
public interface FileUploadService {  
    @Multipart
    @POST("upload")
    Call<ResponseBody> upload(@Part("description") RequestBody description,
                              @Part MultipartBody.Part file);
}
```
如果你需要上传文件，和我们前面的做法类似，定义一个接口方法，需要注意的是，这个方法不再有 @FormUrlEncoded 这个注解，而换成了 @Multipart，后面只需要在参数中增加 Part 就可以了。也许你会问，这里的 Part 和 Field 究竟有什么区别，其实从功能上讲，无非就是客户端向服务端发起请求携带参数的方式不同，并且前者可以携带的参数类型更加丰富，包括数据流。也正是因为这一点，我们可以通过这种方式来上传文件，下面我们就给出这个接口的使用方法：
```java
//先创建 service
FileUploadService service = retrofit.create(FileUploadService.class);
 
//构建要上传的文件
File file = new File(filename);
RequestBody requestFile =
        RequestBody.create(MediaType.parse("application/otcet-stream"), file);
 
MultipartBody.Part body =
        MultipartBody.Part.createFormData("aFile", file.getName(), requestFile);
 
String descriptionString = "This is a description";
RequestBody description =
        RequestBody.create(
                MediaType.parse("multipart/form-data"), descriptionString);
 
Call<ResponseBody> call = service.upload(description, body);
call.enqueue(new Callback<ResponseBody>() {
  @Override
  public void onResponse(Call<ResponseBody> call,
                         Response<ResponseBody> response) {
    System.out.println("success");
  }
 
  @Override
  public void onFailure(Call<ResponseBody> call, Throwable t) {
    t.printStackTrace();
  }
});
```
在实验时，我上传了一个只包含一行文字的文件：
Visit me: http://www.println.net
那么我们去服务端看下我们的请求是什么样的：
HEADERS
```html
Accept-Encoding: gzip
Content-Length: 470
Content-Type: multipart/form-data; boundary=9b670d44-63dc-4a8a-833d-66e45e0156ca
User-Agent: okhttp/3.2.0
X-Request-Id: 9d70e8cc-958b-4f42-b979-4c1fcd474352
Via: 1.1 vegur
Host: requestb.in
Total-Route-Time: 0
Connection: close
Connect-Time: 0
```
FORM/POST PARAMETERS
```html
description: This is a description
```
RAW BODY
```html
--9b670d44-63dc-4a8a-833d-66e45e0156ca
Content-Disposition: form-data; name="description"
Content-Transfer-Encoding: binary
Content-Type: multipart/form-data; charset=utf-8
Content-Length: 21

This is a description
--9b670d44-63dc-4a8a-833d-66e45e0156ca
Content-Disposition: form-data; name="aFile"; filename="uploadedfile.txt"
Content-Type: application/otcet-stream
Content-Length: 32

Visit me: http://www.println.net
--9b670d44-63dc-4a8a-833d-66e45e0156ca--
```
我们看到，我们上传的文件的内容出现在请求当中了。如果你需要上传多个文件，就声明多个 Part 参数，或者试试 PartMap。

### Converter，让你的入参和返回类型丰富起来

#### RequestBodyConverter

前面我为大家展示了如何用 Retrofit 上传文件，这个上传的过程其实。。还是有那么点儿不够简练，我们只是要提供一个文件用于上传，可我们前后构造了三个对象：
<img src="https://raw.githubusercontent.com/way1989/way1989.github.io/hexo/source/postImage/Retrofit/3.jpg"/>
天哪，肯定是哪里出了问题。实际上，Retrofit 允许我们自己定义入参和返回的类型，不过，如果这些类型比较特别，我们还需要准备相应的 Converter，也正是因为 Converter 的存在， Retrofit 在入参和返回类型上表现得非常灵活。
下面我们把刚才的 Service 代码稍作修改：
```java
public interface FileUploadService {  
    @Multipart
    @POST("upload")
    Call<ResponseBody> upload(@Part("description") RequestBody description,
        //注意这里的参数 "aFile" 之前是在创建 MultipartBody.Part 的时候传入的
        @Part("aFile") File file);
}
```
现在我们把入参类型改成了我们熟悉的 File，如果你就这么拿去发请求，服务端收到的结果会让你哭了的。。。

RAW BODY
```html
--7d24e78e-4354-4ed4-9db4-57d799b6efb7
Content-Disposition: form-data; name="description"
Content-Transfer-Encoding: binary
Content-Type: multipart/form-data; charset=utf-8
Content-Length: 21

This is a description
--7d24e78e-4354-4ed4-9db4-57d799b6efb7
Content-Disposition: form-data; name="aFile"
Content-Transfer-Encoding: binary
Content-Type: application/json; charset=UTF-8
Content-Length: 35

// 注意这里！！之前是文件的内容，现在变成了文件的路径
{"path":"samples/uploadedfile.txt"} 
--7d24e78e-4354-4ed4-9db4-57d799b6efb7--
```
服务端收到了一个文件的路径，它肯定会觉得
<img src="https://raw.githubusercontent.com/way1989/way1989.github.io/hexo/source/postImage/Retrofit/4.jpg"/>
好了，不闹了，这明显是 Retrofit 在发现自己收到的实际入参是个 File 时，不知道该怎么办，情急之下给 toString了，而且还是个 JsonString（后来查证原来是使用了 GsonRequestBodyConverter。。）。

接下来我们就自己实现一个 FileRequestBodyConverter，
```java
static class FileRequestBodyConverterFactory extends Converter.Factory {
  @Override
  public Converter<File, RequestBody> requestBodyConverter(Type type, Annotation[] parameterAnnotations, Annotation[] methodAnnotations, Retrofit retrofit) {
    return new FileRequestBodyConverter();
  }
}
 
static class FileRequestBodyConverter implements Converter<File, RequestBody> {
 
  @Override
  public RequestBody convert(File file) throws IOException {
    return RequestBody.create(MediaType.parse("application/otcet-stream"), file);
  }
}
```
在创建 Retrofit 的时候记得配置上它：
```java
addConverterFactory(new FileRequestBodyConverterFactory())
```
这样，我们的文件内容就能上传了。来，看下结果吧：

RAW BODY
```html
--25258f46-48b0-4a6b-a617-15318c168ed4
Content-Disposition: form-data; name="description"
Content-Transfer-Encoding: binary
Content-Type: multipart/form-data; charset=utf-8
Content-Length: 21

This is a description
--25258f46-48b0-4a6b-a617-15318c168ed4
//注意看这里，filename 没了
Content-Disposition: form-data; name="aFile"
//多了这一句
Content-Transfer-Encoding: binary
Content-Type: application/otcet-stream
Content-Length: 32

Visit me: http://www.println.net
--25258f46-48b0-4a6b-a617-15318c168ed4--
```
文件内容成功上传了，当然其中还存在一些问题，这个目前直接使用 Retrofit 的 Converter 还做不到，原因主要在于我们没有办法通过 Converter 直接将 File 转换为 MultiPartBody.Part，如果想要做到这一点，我们可以对 Retrofit 的源码稍作修改，这个我们后面再谈。

#### ResponseBodyConverter

前面我们为大家简单示例了如何自定义 RequestBodyConverter，对应的，Retrofit 也支持自定义 ResponseBodyConverter。

我们再来看下我们定义的接口：
```java
public interface GitHubService {
  @GET("users/{user}/repos")
  Call<List<Repo>> listRepos(@Path("user") String user);
}
```
返回值的类型为 List<Repo>，而我们直接拿到的原始返回肯定就是字符串（或者字节流），那么这个返回值类型是怎么来的呢？首先说明的一点是，GitHub 的这个 api 返回的是 Json 字符串，也就是说，我们需要使用 Json 反序列化得到 List<Repo>，这其中用到的其实是 GsonResponseBodyConverter。

问题来了，如果请求得到的 Json 字符串与返回值类型不对应，比如：

接口返回的 Json 字符串：
```json
{"err":0, "content":"This is a content.", "message":"OK"}
```
返回值类型:
```java
class Result{
    int code;//等价于 err
    String body;//等价于 content
    String msg;//等价于 message
}
```
哇，这时候肯定有人想说，你是不是脑残，偏偏跟服务端对着干？哈哈，我只是示例嘛，而且在生产环境中，你敢保证这种情况不会发生？？
这种情况下， Gson 就是再牛逼，也只能默默无语俩眼泪了，它哪儿知道字段的映射关系怎么这么任性啊。好，现在让我们自定义一个 Converter 来解决这个问题吧！
```java
static class ArbitraryResponseBodyConverterFactory extends Converter.Factory{
  @Override
  public Converter<ResponseBody, ?> responseBodyConverter(Type type, Annotation[] annotations, Retrofit retrofit) {
    return super.responseBodyConverter(type, annotations, retrofit);
  }
}
 
static class ArbitraryResponseBodyConverter implements Converter<ResponseBody, Result>{
 
  @Override
  public Result convert(ResponseBody value) throws IOException {
    RawResult rawResult = new Gson().fromJson(value.string(), RawResult.class);
    Result result = new Result();
    result.body = rawResult.content;
    result.code = rawResult.err;
    result.msg = rawResult.message;
    return result;
  }
}
 
static class RawResult{
  int err;
  String content;
  String message;
}
```
当然，别忘了在构造 Retrofit 的时候添加这个 Converter，这样我们就能够愉快的让接口返回 Result 对象了。

注意！！Retrofit 在选择合适的 Converter 时，主要依赖于需要转换的对象类型，在添加 Converter 时，注意 Converter 支持的类型的包含关系以及其顺序。

## Retrofit 原理剖析

前一个小节我们把 Retrofit 的基本用法和概念介绍了一下，如果你的目标是学会如何使用它，那么下面的内容你可以不用看了。

不过呢，我就知道你不是那种浅尝辄止的人！这一节我们主要把注意力放在 Retrofit 背后的魔法上面~~
<img src="https://raw.githubusercontent.com/way1989/way1989.github.io/hexo/source/postImage/Retrofit/5.jpg"/>

### 是谁实际上完成了接口请求的处理？

前面讲了这么久，我们始终只看到了我们自己定义的接口，比如：
```java
public interface GitHubService {
  @GET("users/{user}/repos")
  Call<List<Repo>> listRepos(@Path("user") String user);
}
```
而真正我使用的时候肯定不能是接口啊，这个神秘的家伙究竟是谁？其实它是 Retrofit 创建的一个代理对象了，这里涉及点儿 Java 的动态代理的知识，直接来看代码：
```java
public <T> T create(final Class<T> service) {
  Utils.validateServiceInterface(service);
  if (validateEagerly) {
    eagerlyValidateMethods(service);
  }
  //这里返回一个 service 的代理对象
  return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
      new InvocationHandler() {
        private final Platform platform = Platform.get();
 
        @Override public Object invoke(Object proxy, Method method, Object... args)
            throws Throwable {
          // If the method is a method from Object then defer to normal invocation.
          if (method.getDeclaringClass() == Object.class) {
            return method.invoke(this, args);
          }
          //DefaultMethod 是 Java 8 的概念，是定义在 interface 当中的有实现的方法
          if (platform.isDefaultMethod(method)) {
            return platform.invokeDefaultMethod(method, service, proxy, args);
          }
          //每一个接口最终实例化成一个 ServiceMethod，并且会缓存
          ServiceMethod serviceMethod = loadServiceMethod(method);
           
          //由此可见 Retrofit 与 OkHttp 完全耦合，不可分割
          OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
          //下面这一句当中会发起请求，并解析服务端返回的结果
          return serviceMethod.callAdapter.adapt(okHttpCall);
        }
      });
}
```
简单的说，在我们调用 GitHubService.listRepos 时，实际上调用的是这里的 InvocationHandler.invoke 方法~~

### 来一发完整的请求处理流程

前面我们已经看到 Retrofit 为我们构造了一个 OkHttpCall ，实际上每一个 OkHttpCall 都对应于一个请求，它主要完成最基础的网络请求，而我们在接口的返回中看到的 Call 默认情况下就是 OkHttpCall 了，如果我们添加了自定义的 callAdapter，那么它就会将 OkHttp 适配成我们需要的返回值，并返回给我们。

先来看下 Call 的接口：
```java
public interface Call<T> extends Cloneable {
  //同步发起请求
  Response<T> execute() throws IOException;
  //异步发起请求，结果通过回调返回
  void enqueue(Callback<T> callback);
  boolean isExecuted();
  void cancel();
  boolean isCanceled();
  Call<T> clone();
  //返回原始请求
  Request request();
}
```
我们在使用接口时，大家肯定还记得这一句：
```java
Call<List<Repo>> repos = service.listRepos("octocat");
List<Repo> data = repos.execute();
```
这个 repos 其实就是一个 OkHttpCall 实例，execute 就是要发起网络请求。
OkHttpCall.execute
```java
@Override public Response<T> execute() throws IOException {
  //这个 call 是真正的 OkHttp 的 call，本质上 OkHttpCall 只是对它做了一层封装
  okhttp3.Call call;
 
  synchronized (this) {
    //处理重复执行的逻辑
    if (executed) throw new IllegalStateException("Already executed.");
    executed = true;
 
    if (creationFailure != null) {
      if (creationFailure instanceof IOException) {
        throw (IOException) creationFailure;
      } else {
        throw (RuntimeException) creationFailure;
      }
    }
     
    call = rawCall;
    if (call == null) {
      try {
        call = rawCall = createRawCall();
      } catch (IOException | RuntimeException e) {
        creationFailure = e;
        throw e;
      }
    }
  }
 
  if (canceled) {
    call.cancel();
  }
  //发起请求，并解析结果
  return parseResponse(call.execute());
}
```
我们看到 OkHttpCall 其实也是封装了 okhttp3.Call，在这个方法中，我们通过 okhttp3.Call 发起了进攻，额，发起了请求。有关 OkHttp 的内容，我在这里就不再展开了。
parseResponse 主要完成了由 okhttp3.Response 向 retrofit.Response 的转换，同时也处理了对原始返回的解析：
```java
Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
  ResponseBody rawBody = rawResponse.body();
 
  //略掉一些代码
  try {
    //在这里完成了原始 Response 的解析，T 就是我们想要的结果，比如 GitHubService.listRepos 的 List<Repo>
    T body = serviceMethod.toResponse(catchingBody);
    return Response.success(body, rawResponse);
  } catch (RuntimeException e) {
    // If the underlying source threw an exception, propagate that rather than indicating it was
    // a runtime exception.
    catchingBody.throwIfCaught();
    throw e;
  }
}
```
至此，我们就拿到了我们想要的数据~~

### 结果适配，你是不是想用 RxJava？

前面我们已经提到过 CallAdapter 的事儿，默认情况下，它并不会对 OkHttpCall 实例做任何处理：
```java
final class DefaultCallAdapterFactory extends CallAdapter.Factory {
  static final CallAdapter.Factory INSTANCE = new DefaultCallAdapterFactory();
 
  @Override
  public CallAdapter<?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
    ... 毫不留情的省略一些代码 ...
    return new CallAdapter<Call<?>>() {
      ... 省略一些代码 ...
 
      @Override public <R> Call<R> adapt(Call<R> call) {
        //看这里，直接把传入的 call 返回了
        return call;
      }
    };
  }
}
```
现在的需求是，我想要接入 RxJava，让接口的返回结果改为 Observable：
```java
public interface GitHub {
  @GET("/repos/{owner}/{repo}/contributors")
  Observable<List<Contributor>> contributors(
      @Path("owner") String owner,
      @Path("repo") String repo);
}
```
可不可以呢？当然是可以的，只需要提供一个 Adapter，将 OkHttpCall 转换为 Observable 即可呀！Retrofit 的开发者们早就想到了这个问题，并且为我们提供了相应的 Adapter：
RxJavaCallAdapterFactory
我们只需要在构造 Retrofit 时，添加它：
```java
addCallAdapterFactory(RxJavaCallAdapterFactory.create())
```
这样我们的接口就可以以 RxJava 的方式工作了。

好，歇会儿，抽一袋烟。。。

接着我们搞清楚 RxJavaCallAdapterFactory 是怎么工作的，首先让我们来看下 CallAdapter 的接口：
```java
public interface CallAdapter<T> {
  /*
   *返回 Http 返回解析后的类型。需要注意的是这个并不是接口的返回类型，
   *而是接口返回类型中的泛型参数的实参。
   */
  Type responseType();
  /*
   * T 是我们需要转换成的接口返回类型，参数 call 其实最初就是 OkHttpCall 的实例
   * 在这里 T 其实是 RxJava 支持的类型，比如 Observable
   */
  <R> T adapt(Call<R> call);
 
  //我们需要将 Factory 的子类对应的实例在构造 Retrofit 时添加到其中。
  abstract class Factory {
    //根据接口的返回类型（Observable<T>），注解类型等等来判断是否是当前 Adapter 支持的类型，不是则返回null
    public abstract CallAdapter<?> get(Type returnType, Annotation[] annotations,
        Retrofit retrofit);
 
    //获取指定 index 的泛型参数的上限，比如对于 Map<String, ? extends Number>，index为 1 的参数上限是 Number
    protected static Type getParameterUpperBound(int index, ParameterizedType type) {
      return Utils.getParameterUpperBound(index, type);
    }
 
    /*
     * 获取原始类型，比如 List<String> 返回 List.class，这里传入的 type 情况可能比较复杂，因此不能直接当做
     * Class 去做判断。这个方法在判断类型是否为支持的类型时经常用到。
    protected static Class<?> getRawType(Type type) {
      return Utils.getRawType(type);
    }
  }
}
```
代码中做了较为详细的注释，简单来说，我们只需要实现 CallAdapter 类来提供具体的适配逻辑，并实现相应的 Factory，用来将当前的 CallAdapter注册到 Retrofit 当中，并在 Factory.get 方法中根据类型来返回当前的 CallAdapter即可。知道了这些，我们再来看 RxJavaCallAdapterFactory：
```java
public final class RxJavaCallAdapterFactory extends CallAdapter.Factory {
 
  ... 请叫我省略君，为了省地方，一个都不放过！ ...
 
  @Override
  public CallAdapter<?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
    //注意下面的代码主要是判断 returnType 是否为 RxJava 支持的类型
    Class<?> rawType = getRawType(returnType);
    String canonicalName = rawType.getCanonicalName();
    boolean isSingle = "rx.Single".equals(canonicalName);
    boolean isCompletable = "rx.Completable".equals(canonicalName);
    if (rawType != Observable.class && !isSingle && !isCompletable) {
      return null;
    }
    ... 这里省略掉的代码主要是根据返回类型获取合适的 Adapter ...
    return callAdapter;
  }
 
  ... 我又来了，继续略去一些代码 ...
 
  static final class SimpleCallAdapter implements CallAdapter<Observable<?>> {
    private final Type responseType;
    private final Scheduler scheduler;
 
    SimpleCallAdapter(Type responseType, Scheduler scheduler) {
      this.responseType = responseType;
      this.scheduler = scheduler;
    }
 
    @Override public Type responseType() {
      return responseType;
    }
 
    @Override public <R> Observable<R> adapt(Call<R> call) {
      //在这里创建需作为返回值的 Observable 实例，并持有 call 实例
      //可以想象得到，在 Observable.subscribe 触发时， call.execute 将会被调用
      Observable<R> observable = Observable.create(new CallOnSubscribe<>(call)) 
          .lift(OperatorMapResponseToBodyOrError.<R>instance());
      if (scheduler != null) {
        return observable.subscribeOn(scheduler);
      }
      return observable;
    }
  }
 
  //... 略去一些代码 ...
}
```
RxJavaCallAdapterFactory 提供了不止一种 Adapter，但原理大同小异，有兴趣的读者可以自行参阅其源码。

至此，我们已经对 CallAdapter 的机制有了一个清晰的认识了。

## 几个进阶玩法

前面我们已经介绍了很多东西了。。可，挖掘机专业的同学们，你们觉得这就够了么？当然是不够！

### 继续简化文件上传的接口

在前面我们曾试图简化文件上传接口的使用，尽管我们已经给出了相应的 File -> RequestBody 的 Converter，不过基于 Retrofit本身的限制，我们还是不能像直接构造 MultiPartBody.Part 那样来获得更多的灵活性。这时候该怎么办？当然是 Hack~~

首先明确我们的需求：
* 文件的 Content-Type 需要更多的灵活性，不应该写死在 Converter 当中，可以的话，最好可以根据文件的扩展名来映射出来对应的 Content-Type， 比如 image.png -> image/png；
* 在请求的数据中，能够正常携带 filename 这个字段。

为此，我增加了一套完整的参数解析方案：
* 增加任意类型转换的 Converter，这一步主要是满足后续我们直接将入参类型转换为 MultiPartBody.Part 类型：

```java
public interface Converter<F, T> {
  ...
   
  abstract class Factory {
    ...
    //返回一个满足条件的不限制类型的 Converter
    public Converter<?, ?> arbitraryConverter(Type originalType,
          Type convertedType, Annotation[] parameterAnnotations, Annotation[] methodAnnotations, Retrofit retrofit){
      return null;
    }
  }
}
```
需要注意的是，Retrofit 类当中也需要增加相应的方法：
```java
public <F, T> Converter<F, T> arbitraryConverter(Type orignalType,
                                                 Type convertedType, Annotation[] parameterAnnotations, Annotation[] methodAnnotations) {
  return nextArbitraryConverter(null, orignalType, convertedType, parameterAnnotations, methodAnnotations);
}
 
public <F, T> Converter<F, T> nextArbitraryConverter(Converter.Factory skipPast,
                              Type type, Type convertedType,  Annotation[] parameterAnnotations, Annotation[] methodAnnotations) {
  checkNotNull(type, "type == null");
  checkNotNull(parameterAnnotations, "parameterAnnotations == null");
  checkNotNull(methodAnnotations, "methodAnnotations == null");
 
  int start = converterFactories.indexOf(skipPast) + 1;
  for (int i = start, count = converterFactories.size(); i < count; i++) {
    Converter.Factory factory = converterFactories.get(i);
    Converter<?, ?> converter =
            factory.arbitraryConverter(type, convertedType, parameterAnnotations, methodAnnotations, this);
    if (converter != null) {
      //noinspection unchecked
      return (Converter<F, T>) converter;
    }
  }
  return null;
}
```

* 再给出 arbitraryConverter 的具体实现：

```java
public class TypedFileMultiPartBodyConverterFactory extends Converter.Factory {
    @Override
    public Converter<TypedFile, MultipartBody.Part> arbitraryConverter(Type originalType, Type convertedType, Annotation[] parameterAnnotations, Annotation[] methodAnnotations, Retrofit retrofit) {
        if (originalType == TypedFile.class && convertedType == MultipartBody.Part.class) {
            return new FileRequestBodyConverter();
        }
        return null;
    }
}
```
```java
public class TypedFileMultiPartBodyConverter implements Converter<TypedFile, MultipartBody.Part> {
 
    @Override
    public MultipartBody.Part convert(TypedFile typedFile) throws IOException {
        RequestBody requestFile =
                RequestBody.create(typedFile.getMediaType(), typedFile.getFile());
        return MultipartBody.Part.createFormData(typedFile.getName(), typedFile.getFile().getName(), requestFile);
    }
}
```
```java
public class TypedFile {
    private MediaType mediaType;
    private String name;
    private File file;
 
    public TypedFile(String name, String filepath){
        this(name, new File(filepath));
    }
 
    public TypedFile(String name, File file) {
        this(MediaType.parse(MediaTypes.getMediaType(file)), name, file);
    }
 
    public TypedFile(MediaType mediaType, String name, String filepath) {
        this(mediaType, name, new File(filepath));
    }
 
    public TypedFile(MediaType mediaType, String name, File file) {
        this.mediaType = mediaType;
        this.name = name;
        this.file = file;
    }
 
    public String getName() {
        return name;
    }
 
    public MediaType getMediaType() {
        return mediaType;
    }
 
    public File getFile() {
        return file;
    }
}
```

* 在声明接口时，@Part 不要传入参数，这样 Retrofit 在 ServiceMethod.Builder.parseParameterAnnotation 方法中解析 Part时，就会认为我们传入的参数为 MultiPartBody.Part 类型（实际上我们将在后面自己转换）。那么解析的时候，我们拿到前面定义好的 Converter，构造一个 ParameterHandler：

```java
...
} else if (MultipartBody.Part.class.isAssignableFrom(rawParameterType)) {
    return ParameterHandler.RawPart.INSTANCE;
} else {
    Converter<?, ?> converter =
            retrofit.arbitraryConverter(type, MultipartBody.Part.class, annotations, methodAnnotations);
    if(converter == null) {
        throw parameterError(p,
                "@Part annotation must supply a name or use MultipartBody.Part parameter type.");
    }
    return new ParameterHandler.TypedFileHandler((Converter<TypedFile, MultipartBody.Part>) converter);
}
...
```
```java
static final class TypedFileHandler extends ParameterHandler<TypedFile>{
 
  private final Converter<TypedFile, MultipartBody.Part> converter;
 
  TypedFileHandler(Converter<TypedFile, MultipartBody.Part> converter) {
    this.converter = converter;
  }
 
  @Override
  void apply(RequestBuilder builder, TypedFile value) throws IOException {
    if(value != null){
      builder.addPart(converter.convert(value));
    }
  }
}
```
* 这时候再看我们的接口声明：

```java
public interface FileUploadService {
  @Multipart
  @POST("upload")
  Call<ResponseBody> upload(@Part("description") RequestBody description,
                            @Part TypedFile typedFile);
}
```
以及使用方法：
```java
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("http://www.println.net/")
    .addConverterFactory(new TypedFileMultiPartBodyConverterFactory())
    .addConverterFactory(GsonConverterFactory.create())
    .build();
 
FileUploadService service = retrofit.create(FileUploadService.class);
TypedFile typedFile = new TypedFile("aFile", filename);
String descriptionString = "This is a description";
RequestBody description =
        RequestBody.create(
                MediaType.parse("multipart/form-data"), descriptionString);
 
Call<ResponseBody> call = service.upload(description, typedFile);
call.enqueue(...);
```
至此，我们已经通过自己的双手，让 Retrofit 的点亮了自定义上传文件的技能，风骚等级更上一层楼！

### Mock Server

我们在开发过程中，经常遇到服务端不稳定的情况，测试开发环境，这是难免的。于是我们需要能够模拟网络请求来调试我们的客户端逻辑，Retrofit 自然是支持这个功能的。

真是太贴心，Retrofit 提供了一个 MockServer 的功能，可以在几乎不改动客户端原有代码的前提下，实现接口数据返回的自定义，我们在自己的工程中增加下面的依赖：
```groovy
compile 'com.squareup.retrofit2:retrofit-mock:2.0.2
```
还是先让我们来看看官方 demo，首先定义了一个 GituHb api，好熟悉的感觉：
```java
public interface GitHub {
  @GET("/repos/{owner}/{repo}/contributors")
  Call<List<Contributor>> contributors(
      @Path("owner") String owner,
      @Path("repo") String repo);
}
```
这就是我们要请求的接口了，怎么 Mock 呢？
* 定义一个接口实现类 MockGitHub，我们可以看到，所有我们需要请求的接口都在这里得到了实现，也就是说，我们待会儿调用 GitHub 的 api 时，实际上是访问 MockGitHub 的方法：

```java
static final class MockGitHub implements GitHub {
    private final BehaviorDelegate<GitHub> delegate;
    private final Map<String, Map<String, List<Contributor>>> ownerRepoContributors;
 
    public MockGitHub(BehaviorDelegate<GitHub> delegate) {
      this.delegate = delegate;
      ownerRepoContributors = new LinkedHashMap<>();
 
      // Seed some mock data.
      addContributor("square", "retrofit", "John Doe", 12);
      addContributor("square", "retrofit", "Bob Smith", 2);
      addContributor("square", "retrofit", "Big Bird", 40);
      addContributor("square", "picasso", "Proposition Joe", 39);
      addContributor("square", "picasso", "Keiser Soze", 152);
    }
 
    @Override public Call<List<Contributor>> contributors(String owner, String repo) {
      List<Contributor> response = Collections.emptyList();
      Map<String, List<Contributor>> repoContributors = ownerRepoContributors.get(owner);
      if (repoContributors != null) {
        List<Contributor> contributors = repoContributors.get(repo);
        if (contributors != null) {
          response = contributors;
        }
      }
      return delegate.returningResponse(response).contributors(owner, repo);
    }
 
    public void addContributor(String owner, String repo, String name, int contributions) {
      Map<String, List<Contributor>> repoContributors = ownerRepoContributors.get(owner);
      if (repoContributors == null) {
        repoContributors = new LinkedHashMap<>();
        ownerRepoContributors.put(owner, repoContributors);
      }
      List<Contributor> contributors = repoContributors.get(repo);
      if (contributors == null) {
        contributors = new ArrayList<>();
        repoContributors.put(repo, contributors);
      }
      contributors.add(new Contributor(name, contributions));
    }
}
```

* 构建 Mock Server 对象：

```java
// Create a very simple Retrofit adapter which points the GitHub API.
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl(SimpleService.API_URL)
    .build();
 
// Create a MockRetrofit object with a NetworkBehavior which manages the fake behavior of calls.
NetworkBehavior behavior = NetworkBehavior.create();
MockRetrofit mockRetrofit = new MockRetrofit.Builder(retrofit)
    .networkBehavior(behavior)
    .build();
 
BehaviorDelegate<GitHub> delegate = mockRetrofit.create(GitHub.class);
MockGitHub gitHub = new MockGitHub(delegate);
```

* 使用 Mock Server ：

```java
Call<List<Contributor>> contributors = gitHub.contributors(owner, repo);
...
```
也就是说，我们完全可以自己造一个假的数据源，通过 Mock Server 来返回这些写数据。

那么问题来了，这其实并没有完全模拟网络请求的解析流程，如果我只能提供原始的 json 字符串，怎么通过 Retrofit 来实现 Mock Server？

时间已经不早啦，我就不猥琐发育了，直接推塔~

本文前面一直专注于介绍 Retrofit，很少提及 OkHttp，殊不知 OkHttp 有一套拦截器的机制，也就是说，我们可以任性的检查 Retrofit 即将发出或者正在发出的所有请求，并且篡改它。所以我们只需要找到我们想要的接口，定制自己的返回结果就好了，下面是一段示例：
```java
OkHttpClient client = new OkHttpClient.Builder().addInterceptor(new Interceptor() {
    @Override
    public Response intercept(Chain chain) throws IOException {
        Response response = null;
        if(BuildConfig.DEBUG && chain.request().url().uri().getPath().equals("/contributors")) {
            //这里读取我们需要返回的 Json 字符串
            String responseString = ...;
             
            response = new Response.Builder()
                    .code(200)
                    .message(responseString)
                    .request(chain.request())
                    .protocol(Protocol.HTTP_1_0)
                    .body(ResponseBody.create(MediaType.parse("application/json"), responseString.getBytes()))
                    .addHeader("content-type", "application/json")
                    .build();
        } else {
            response = chain.proceed(chain.request());
        }
 
        return response;
    }
}).build();
 
Retrofit retrofit = new Retrofit.Builder()
        .baseUrl(BASE_URL)
        .addConverterFactory(GsonConverterFactory.create())
        .client(client)
        .build();
```
这样，我们就会拦截 contributors 这个 api 并定制其返回了。

## 小结

Retrofit 是非常强大的，本文通过丰富的示例和对源码的挖掘，向大家展示了 Retrofit 自身强大的功能以及扩展性，就算它本身功能不能满足你的需求，你也可以很容易的进行改造，毕竟人家的代码真是写的漂亮啊。