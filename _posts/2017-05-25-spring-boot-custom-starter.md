---
layout:     post
title:      "自定义SpringBoot Starter!"
subtitle:   "Custom SpringBoot Starter"
date:       2017-05-25
author:     "leasy"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - Spring
    - Java
    - SpringBoot
---

> "Use Spring from now on"

SpringBoot 和传统Spring项目的区别之一就是自动配置，自动配置是依赖于各种starter实现，比如spring-boot-starter-logging等等，为了了解SpringBoot starter的工作原理，我尝试着自己写一个自定义的starter.

#### SpringBoot starter实现流程
首先了解现有的一些库如何使用SpringBoot starter来实现自动配置，以springboot-mybatis-starter为例子。springboot-mybatis-starter项目的目录结构如下
![目录结构](https://leasyzhang.github.io/img/index-springboot-starter.jpg)
在autoconfigure项目的src/main/resources/META-INF目录下，有一个spring.factories文件，里面有这一行

```java
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.mybatis.spring.boot.autoconfigure.MybatisAutoConfiguration
```

这一行表示SpringBoot项目启动的时候如果启动了自动配置就会加载MybatisAutoConfiguration这个类，这个类的作用是加载mybatis相关的配置然后使用mybatis提供的功能，在类的开始用注解

```java
@ConditionalOnClass({ SqlSessionFactory.class, SqlSessionFactoryBean.class })
@ConditionalOnBean(DataSource.class)
@EnableConfigurationProperties(MybatisProperties.class)
@AutoConfigureAfter(DataSourceAutoConfiguration.class)
```
@ConditionalOnClass 表示如果当前项目的classpath中有SqlSessionFactory和SqlSessionFactoryBean两个累的时候会使用自动配置，@ConditionalOnBean 表示这个Bean必须定义之后才有效。然后在starter项目的pom文件中引入autoconfigure和mybatis等库，我们在使用的时候只需要引入mybatis-starter就可以了。
#### 编写自己的starter
大致了解流程之后，可以自己写自己的starter了。项目代码在[这里](https://github.com/LeasyZhang/spring-boot-custom-starter)
项目目录
![springboot-my-starter项目目录](https://leasyzhang.github.io/img/index-springboot-starter.jpg)
#### MyInfo项目
这个项目下面有三个类，MyInfo(读取配置文件的数据然后在控制台输出这些值)，MyInfoConfig(继承Properties类，用来读取.properties配置文件)，MyInfoConfigKey(存储配置文件的key值，key值对应配置文件的key值，这是约定好的)这个项目是我们要用到的，接下来要做的事情是把这个项目引入到springboot里面去
#### MyInfoAutoConfiguration
首先在src/resources目录下建立META-INF文件夹，然后在里面创建spring.factories文件，在里面写上
```bash
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
MyInfoAutoConfiguration
```
然后在对应的路径下面建立MyInfoAutoConfiguration这个类，用@Configuration 这个注解标注这个类，这样SpringBoot在启动的时候可以注入这个类，这个类做的事情是将MyInfo项目的MyInfo类和MyInfoConfig类用@Bean注解标示。还有就是执行具体的读取配置文件的操作，用户在使用的时候只需要配置一下配置文件，至于配置文件的数据如何赋值给MyInfo，是AutoConfigure做的事情。
#### MyInfo Starter
这个项目没有java代码，就一个pom文件，将MyInfo和MyInfoAutoConfiguration项目的依赖添加进去，用户用的时候只需要引入MyInfo-Starter即可。
#### MyInfo Sample
这是测试用的项目，测试代码如下

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class MyInfoApplication implements CommandLineRunner {

    @Autowired
    private MyInfo myInfo;

    public void run(String... args) throws Exception {
        System.out.println(myInfo.print());
    }

    public static void main(String[] args) {
        SpringApplication.run(MyInfoApplication.class, args);
    }
}
```

![配置文件](https://leasyzhang.github.io/img/index-springboot-starter-config.jpg)

#### 总结
创建一个自定义的SpringBoot starter通常需要一下步骤
- 创建spring.factories文件，为EnableAutoConfiguration制定一个类 CustomAuthConfigurationClass
- 创建类CustomAuthConfigurationClass，加上注解@Configuration(必填)，还可以加上一些可选注解@ConditionalOnClass等等，表示当前的类加载之前需要等待其它类加载
- 创建一个空项目starter，只有一个POM文件，里面依赖CustomAuthConfigurationClass所在的模块
- 用maven install命令打包成一个库，上传到maven仓库
- 依赖starter对应的库，就可以在SpringBoot项目启动的时候自动运行CustomAuthConfigurationClass了。