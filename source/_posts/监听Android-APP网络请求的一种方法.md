---
title: 监听Android APP网络请求的一种方法
date: 2017-03-26 19:26:19
tags:
---

目前市面上有很多APM厂商，用户只用集成Android SDK，写一行启动代码就可以轻松实现对APP网络请求的监听，非常神奇。本文就介绍一种可以实现该功能的技术，实现对APP网络请求的全量监听。

## 需求
1. SDK集成后能实现对APP网络请求的监听。
2. 接入无成本，只需要少量的启动代码，不需要修改已有的代码。

## 可以实现监听网络请求的技术方案
##### 1. 实现整个http网络框架，让用户去调用。
该方案实现成本过高，而且接入成本也很高，基本不能实现。
##### 2. Android热修复方案
目前以Dexposed为代表的热修复方案无法支持ART虚拟机，不能用于5.0以上的Android操作系统；以Nova为代表的热修复方案需要首先被加载并且无法对修改现有类进行修改，不能用于sdk。
##### 3. Java AOP编程技术
该方案能成功实现对Java的Hook，而且灵活可控，能够有效实现对API的监听。
#### 现有Java AOP技术
1. Apt
2. Aspectj
3. Javassit

它们的区别如下图所示
![img](/img/监听Android-APP网络请求的一种方法_1.png)

本文采用Aspectj来实现网络请求监听SDK
## Android Aspectj插件
[Android Aspectj Git Repo](https://github.com/HujiangTechnology/gradle_plugin_android_aspectjx)
##### 接入步骤如下所示：
1. 关闭Android Intant Run
![img](/img/close_instant_run.png)
2. 修改Project的.gradle文件
![img](/img/add_config_to_project.jpg)
3. 修改Module的.gradle文件
![img](/img/add_config_to_module.jpg)

## API Hook语法介绍
常用的annotation有Before、Around、After，分别是在目标函数(需要进行aspectj hook的函数)执行的前、中、后期进行hook。主要的hook方式分为call和execution两种，它们区别如下所示:
**call**
``` js
// <------- before call JoinPoint 
targetFunc() // <------ around call JoinPoint
// <------- after call JoinPoint 
```
**exection**
``` js
targetFunc() {
    // <------- before execution JoinPoint 
    ... // <------ around exection JoinPoint
    // <------- after exection JoinPoint
}
```
下面将通过一段代码来演示：
**Activity.java**
```
public class MainActivity extends AppCompatActivity {
    private static final String TAG = "MainActivity";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        new Example().SayHelloWord();
    }
}
```
**Example.java**
```
public class Example {
    private static final String TAG = "Example";

    public void SayHelloWord() {
        Log.i(TAG, "Hello world!");
    }
}
```
**AspecjExample.java**
```
@Aspect
public class AspecjExample {
    private static final String TAG = "AspecjExample";
    protected static final AspecjExample instance = new AspecjExample();

    @Before("call(* testapp.aspectjtest.Example.SayHelloWord(..))")
    public void BeforeCall(JoinPoint joinPoint) throws Throwable {
        Log.i(TAG, "Aspectj Before call SayHelloWord");
    }

    @Before("execution(* testapp.aspectjtest.Example.SayHelloWord(..))")
    public void BeforeExecution(JoinPoint joinPoint) throws Throwable {
        Log.i(TAG, "Aspectj Before execution SayHelloWord");
    }

    @Around("call(* testapp.aspectjtest.Example.SayHelloWord(..))")
    public Object AroundCall(ProceedingJoinPoint joinPoint) throws Throwable {
        Log.i(TAG, "Aspectj Around start call SayHelloWord");
        Object ret = joinPoint.proceed();
        Log.i(TAG, "Aspectj Around end call SayHelloWord");
        return ret;
    }

    @Around("execution(* testapp.aspectjtest.Example.SayHelloWord(..))")
    public Object AroundExecution(ProceedingJoinPoint joinPoint) throws Throwable {
        Log.i(TAG, "Aspectj Around start execution SayHelloWord");
        Object ret = joinPoint.proceed();
        Log.i(TAG, "Aspectj Around end execution SayHelloWord");
        return ret;
    }

    @After("call(* testapp.aspectjtest.Example.SayHelloWord(..))")
    public void AfterCall(JoinPoint joinPoint) throws Throwable {
        Log.i(TAG, "Aspectj After call SayHelloWord");
    }

    @After("execution(* testapp.aspectjtest.Example.SayHelloWord(..))")
    public void AfterExecution(JoinPoint joinPoint) throws Throwable {
        Log.i(TAG, "Aspectj After execution SayHelloWord");
    }

    public static AspecjExample aspectOf() {
        return instance;
    }
}
```
输入结果为：
![img](/img/aspectj_example_output.png)

## 如何实现对网络请求API的监听
#### 监听URLConnection:
```
@Around("call(* java.net.URL+.openConnection(..))")
public Object onHttpURLOpenConnect(ProceedingJoinPoint joinPoint) throws Throwable {
    ... // 建立Http连接，并获取主机IP和DNS时间
}

@Around("call(* java.net.URLConnection+.getInputStream(..))")
public Object onHttpURLConnectInput(ProceedingJoinPoint joinPoint) throws Throwable {
    ... // 获取InputSteam，创建该InputSteam的代理，并且将代理作为结果返回
}
```

#### 监听OkHttp:
```
@Around("call(* okhttp3.Dns+.lookup(..))")
public Object onOkHttp3DnsLookup(ProceedingJoinPoint joinPoint) throws Throwable {
    ... // 获取Host IP和DNS时间
}

@Around("call(* java.net.InetSocketAddress+.createUnresolved(..))")
public Object onSocketAddressResolve(ProceedingJoinPoint joinPoint) throws Throwable {
    ... // 获取Host IP和DNS时间
}

@Around("call(* okhttp3.Response.Builder+.build(..))")
public Object onOkHttp3RespBuild(ProceedingJoinPoint joinPoint) throws Throwable {
    ... // 保存Response和ResponseBody
}

@Around("call(* okhttp3.ResponseBody+.source(..))")
public Object onOkHttp3RespBodySource(ProceedingJoinPoint joinPoint) throws Throwable {
    ... // 建立ResponseBody的代理，并把该代理作为结果返回
}
```

#### 监听WebView
```
@Around("call(* android.webkit.WebView+.setWebViewClient(..))")
public Object onWebViewSetClient(ProceedingJoinPoint joinPoint) throws Throwable {
    ... // 建立WebViewClient代理，并把带监听功能的代理作为结果返回
}

@Before("call(* android.webkit.WebView+.loadUrl(..))")
public void onWebViewLoadUrl(JoinPoint joinPoint) {
    ... // 如果该WebView未设置WebViewClient，就给它设置监听WebViewClient
}
```
