---
title: Spring小记之Bean
comments: true
date: 2018-04-04 20:41:27
categories:
- Spring
tags:
- Spring
- Java
---

本次主要讲点与Bean有关的常用配置。

## Bean的Scope

Scope也就是作用域，描述Spring容器创建Bean实例的过程。有以下几种，使用@Scope注解来实现。

1. Singleton: 一个Spring容器只有一个Bean实例，全Spring容器共享一个实例，此为默认配置。
2. Prototype:  每次调用新创建一个实例。
3. Request: Web项目中，每个request创建一个实例。
4. Session: 同上，每个Session一个实例。

**示例:**

```java
// 默认为Singleton
@Scope("prototype")
@Component()
public class Song {
	// ...
}
```

## Spring EL和资源调用

Spring开发中经常涉及到调用各种资源的情况，包含普通文件，网址，配置文件，系统环境变量等。我们可以使用Spring的 EL来实现资源的注入。

直接看示例

1. 在resrouces文件夹下创建props/test.properties, 并添加

    ```properties
   author.name=marlondu
    ```

2. 创建java config类

```java
package com.adu.config;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.PropertySource;
import org.springframework.core.env.Environment;

/**
 * @author aduchuan@126.com
 * @date 2018/4/4
 * @since 1.0
 */
@Configuration
@PropertySource("classpath:props/test.properties")
@ComponentScan("com.adu")
public class ElConfig {

    @Value("I Love you")
    private String normal;

    @Value("#{systemProperties['os.name']}")
    private String osName;

    @Value("#{T(java.lang.Math).random() * 100.0}")
    private Double randomNumber;

    @Value("#{anotherService.name}")
    private String fromAnother;

    @Value("${author.name}")
    private String authorName;

    @Autowired
    private Environment environment;

    /*@Bean
    public static PropertySourcesPlaceholderConfigurer propertyConfigure() {
        return new PropertySourcesPlaceholderConfigurer();
    }*/

    public void outputResource() {
        System.out.println(normal);
        System.out.println(osName);
        System.out.println(randomNumber);
        System.out.println(fromAnother);
        System.out.println(authorName);
        System.out.println(environment.getProperty("author.name"));
    }
}

package com.adu.service;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

/**
 * @author aduchuan@126.com
 * @date 2018/4/4
 * @since 1.0
 */
@Service
public class AnotherService {

    @Value("another service")
    private String name;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

3. 运行程序

   ```java
   package com.adu.run;

   import com.adu.config.ElConfig;
   import org.springframework.context.annotation.AnnotationConfigApplicationContext;

   /**
    * @author duchuanchuan
    * @date 2017/1/13
    */
   public class Demo {
       public static void main(String[] args) {
           final AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(ElConfig.class);
           ElConfig elConfig = context.getBean(ElConfig.class);
           elConfig.outputResource();
           context.close();
       }
   }
   ```

   ​

## Bean的初始化和销毁

在实际开发的时候，经常会遇到需要在Bean创建后和销毁后需要做些必要的操作。Spring对Bean的生命周期的操作提供了支持。有两种方式来使用。

- Java配置方式： 使用@Bean注解的initMethod 和destroyMethod(相当于xml配置中的init-method和destroy-method)
- 注解方式： 利用JSR-250的@PostConstruct和@PreDestroy
  - @PostConstruct: 在构造函数执行完之后执行
  - @PreDestroy: 在Bean销毁之前执行

一般使用注解方式：

```java
@Service
public class AnotherService {

    @Value("another service")
    private String name;

    @PostConstruct
    private void initMethod() {
		// do something
    }

    @PreDestroy
    public void preDestroy() {
		// do something
    }
}
```



## 事件（Application Event）

Spring的事件为Bean与Bean之间的消息通信提供了支持。当一个bean处理完一个任务后，希望另一个bean知道并能做相应的处理，这时候我们就需要另一个Bean来监听这个bean发送的事件。以前不知道还有这种操作。

Spring的事件遵循一下流程：

1. 自定义事件，继承ApplicationEvent.
2. 定义事件监听器， 实现ApplicationListener.
3. 使用容器发布事件。

示例：

```java
//1. 定义事件类
public class MessageEvent extends ApplicationEvent{

    private String msg;
    public MessageEvent(Object source, String msg) {
        super(source);
        this.msg = msg;
    }

    public String getMsg() {
        return msg;
    }

    public void setMsg(String msg) {
        this.msg = msg;
    }
}
//2. 定义监听类
@Component
public class MessageListener implements ApplicationListener<MessageEvent> {

    @Override
    public void onApplicationEvent(MessageEvent event) {
        String msg = event.getMsg();
        System.out.println("I(Message Listener) accepted message from publisher: " + msg);
    }
}
//3.定义发布类
@Component
public class MessagePublisher {
    @Autowired
    private ApplicationContext context;

    public void publish(String msg) {
        context.publishEvent(new MessageEvent(this, msg));
    }
}
//4. 运行程序
public class Demo {
    public static void main(String[] args) {
        final AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(ElConfig.class);
        MessagePublisher publisher = context.getBean(MessagePublisher.class);
        publisher.publish("hello, nice to meet you");
        context.close();
    }
}
//5.运行结果
// I(Message Listener) accepted message from publisher: hello, nice to meet you
```

Application Event真的时超级简单，在实际应用中可以尝试一下。在某些需要协同服务时，应该是有很大应用空间的。