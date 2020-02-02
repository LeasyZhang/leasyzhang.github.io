---
layout:     post
title:      "SpringBoot项目使用异步任务"
subtitle:   "@Async 注解使用"
date:       2020-02-02
author:     "leasy"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - SpringBoot
    - Java
    - 多线程
---

> "Spring Async功能使用"

最近在项目中需要实现一些统计的Job，因为这些Job比较耗时，而且项目还有一些Controller来接收客户端请求，每次Job执行的时候，如果这时候有请求过来，请求就会卡住直到job完成，这个体验非常糟糕。
考虑到每个Job都是独立运行的没有被其他地方依赖，所以打算将这些Job用异步调用的方式来实现。
项目是SpringBoot项目，经过调查发现Spring框架提供了@Async的注解，这个注解可以帮助我们很轻松的实现异步调用的功能。

#### 依赖

- Gradle 6.0.1
- SpringBoot 2.2.2.RELEASE
- JDK 8

#### 使用Async注解

Async注解的使用非常方便

- 首先在启动类加上@EnableAsync注解

```java
@SpringBootApplication
@EnableAsync
public class Application {

    public static void main(String[] args) {
        SpringApplication.Run(Application.class, args);
    }
}
```

- 在方法上加上@Async注解

```java
@Component
public class SampleJob {

    @Async
    public void calculate() {
        //这里做一些耗时的事情
    }

    //带有返回值的异步方法
    @Async
    public Future<String> calAndReturn() {
        return new AsyncResult<String>(String.valueOf("result"));
    }
}
```

这样子calculate这个方法就会被放到其他线程中执行。

#### 自定义线程池

Spring框架会创建一个默认的线程池执行所有Async注解标记的方法。如果我们想要使用自定义的线程池，可以创建一个ThreadPoolTaskExecutor类，然后设置线程池参数。

- 创建一个线程池配置类

```java
@Configuration
@EnableAsync
public class ThreadPoolConfig {

    @Bean("customThreadPool")
    public ThreadPoolTaskExecutor customThreadPool() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();

        //设定参数
        executor.setCorePoolSize(4);
        //...设定其他参数
        //初始化
        executor.initialize();
        return executor;
    }
}
```

- 移除入口类上的@EnableAsync注解

```java
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.Run(Application.class, args);
    }
}
```

- 在Async注解上指定线程池名字

```java
@Component
public class SampleJob {

    @Async(value = "customThreadPool")
    public void calculate() {
        //这里做一些耗时的事情
    }

    //带有返回值的异步方法
    @Async(value = "customThreadPool")
    public Future<String> calAndReturn() {
        return new AsyncResult<String>(String.valueOf("result"));
    }
}
```

这样子就可以使用我们指定的线程池来执行异步任务。

#### 定义异常处理器

Async注解的方法如果出现异常，可以使用AsyncConfigure来定义全局异常处理器。

```java
@Configuration
public class AsyncExceptionConfig implements AsyncConfigurer {

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return new CustomExceptionHandler();
    }

    class CustomExceptionHandler implements AsyncUncaughtExceptionHandler {
        @Override
        public void handleUncaughtException(Throwable throwable, Method method, Object... objects) {
            //处理异常
        }
    }
}
```

线程里面的方法如果出现了异常，会被handleUncaughtException方法捕获，我们可以在这里记录和处理异常。

#### 总结

这篇文章介绍了Spring Async注解的用法，以及如何配置自定义的线程池和异常处理器来执行任务和处理错误。
