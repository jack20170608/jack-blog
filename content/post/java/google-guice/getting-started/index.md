---
title: "Google guice系列(二)  快速入门"
categories: ["Java", "guice"]
tags: ["guice","IOC","依赖注入"]
date: 2021-02-28T15:21:23+08:00
draft: false
toc: true
---


注意：这篇文章主要翻译自guice的github [guice-GettingStarted](https://github.com/google/guice/wiki/GettingStarted)


## 快速入门

Guice框架能够更容易帮助你的应用程序实现依赖注入(DI)。这篇入门指南将通过一个简单的示例来引导你怎么使用Guice，把依赖注入整合到你的应用程序中。

## 什么是依赖注入
&nbsp;&nbsp;依赖注入是一种设计模式，其中类将其依赖项声明为参数，而不是由类本身直接创建那些依赖项。举个例子：一个客户端想要调用一个服务，它不用知道怎么创建这个服务，相反这个任何是由一些外部的代码来完成的。
下面是一个不使用依赖注入的简单的示例代码:
```java
class Foo {
  private Database database;  // We need a Database to do some work

  Foo() {
    // Ugh. How could I test this? What if I ever want to use a different
    // database in another application?
    this.database = new Database("/path/to/my/data");
  }
}
```

&nbsp;&nbsp;例子中Foo类直接创建了一个`DataBase`类instance，这就固定死了Foo类只能使用自己创建的Database，而且在单元测试代码中，无法用测试数据库代替真实数据库。但是可以使用依赖注入模式来解决无法单元测试以及代码不灵活的问题。下面使用了依赖注入后的Foo类：
```java
class Foo {
  private Database database;  // We need a Database to do some work

  // The database comes from somewhere else. Where? That's not my job, that's
  // the job of whoever constructs me: they can choose which database to use.
  Foo(Database database) {
    this.database = database;
  }
}
```
&nbsp;&nbsp; 上面的Foo类就能够使用任何的Database对象，因为Foo完全不知道Database是如何创建的。例如，你可以创建一个只用于测试的内存数据库的Database实例，这能够使测试更快速和完整。
在[开发动机](../motivation)这篇文章中对为什么要使用依赖注入有更多解释。

## Guice的核心概念

&nbsp;&nbsp; 在Java类中申明了`@Inject`注解的构造函数，将被Guice的一个名为constructor injection的线程所调用，在调用过程中，构造函数的参数将由Guice所提供。下面是一个使用constructor injection的示例：
```java
class Greeter {
  private final String message;
  private final int count;

  // Greeter declares that it needs a string message and an integer
  // representing the number of time the message to be printed.
  // The @Inject annotation marks this constructor as eligible to be used by
  // Guice.
  @Inject
  Greeter(@Message String message, @Count int count) {
    this.message = message;
    this.count = count;
  }

  void sayHello() {
    for (int i=0; i < count; i++) {
      System.out.println(message);
    }
  }
}
```
&nbsp;&nbsp;在上面这个例子中，当应用程序让Guice去创建一个`Greeter`实例时，Guice将创建该构造方法的两个参数，并用他们作为构造方法的参数去调用。`Greeter`的构造方法参数作为它自己的依赖项，并且应用程序使用`Module`模块来告诉Guice如何满足他的依赖项。

## Guice Modules
&nbsp;&nbsp;应用程序对象通过声明对其他对象的依赖关系，这种依赖关系会行成一种图的数据结构，例如上面的例子`Greeter`类有2个依赖项(在构造方法中声明的)。
1. 一个`String`类型的对象用于消息打印
2. 一个`Integer`类型的对象用于消息打印的次数
   
&nbsp;&nbsp;Guice modules允许应用程序自己指定如何满足对象间的依赖关系。例如：下面的`DemoModule`为`Greeter`类配置了所有必须的依赖项。
```java
/**
 * Guice module that provides bindings for message and count used in
 * {@link Greeter}.
 */
import com.google.inject.Provides;

class DemoModule extends AbstractModule {
    @Provides
    @Count
    static Integer provideCount() {
        return 3;
    }

    @Provides
    @Message
    static String provideMessage() {
        return "hello world";
    }
}
```

`DemoModule`使用了2种Guice支持的方式。
* @Provides method
* Guice自己的领域语言，使用`AbstractModule`中的各种绑定方法
  
&nbsp;&nbsp;在实际的应用程序中，对象依赖图像更加复杂，但是Guice自动的使得整个创建过程更加容易，包括各种传递的依赖关系。

## Guice injectors 
&nbsp;&nbsp; 你将需要创建一个包含一个或者多个module的Guice`Injector`来引导应用程序。例如，下面的一个web的应用程序程序，在main方法中创建了一个`Injector`: 
```java
public final class MyWebServer {
    public void start() {
        ...
    }

    public static void main(String[] args) {
        // Creates an injector that has all the necessary dependencies needed to
        // build a functional server.
        Injector injector = Guice.createInjector(
            new RequestLoggingModule(),
            new RequestHandlerModule(),
            new AuthenticationModule(),
            new DatabaseModule(),
            ...);
        // Bootstrap the application by creating an instance of the server then
        // start the server to handle incoming requests.
        injector.getInstance(MyWebServer.class)
            .start();
    }
}
```
&nbsp;&nbsp;Injector内部维护了应用程序的对象依赖关系图。当需要一个指定类型的对象的实例，injector将找出并构造出相应的对象，解析他们的依赖关系，并将对象们连接在一起。要指定如何解析依赖项，可以通过`binding`来配置injector。

## 一个简单的Guice应用
&nbsp;&nbsp;下面是一个简单的Guice应用实例包含，包括所有需要的部分。
```java
package guicedemo;

import static java.lang.annotation.RetentionPolicy.RUNTIME;

import com.google.inject.AbstractModule;
import com.google.inject.Guice;
import com.google.inject.Injector;
import com.google.inject.Key;
import com.google.inject.Provides;
import java.lang.annotation.Retention;
import javax.inject.Inject;
import javax.inject.Qualifier;

public class GuiceDemo {
    @Qualifier
    @Retention(RUNTIME)
    @interface Message {}

    @Qualifier
    @Retention(RUNTIME)
    @interface Count {}

    /**
    * Guice module that provides bindings for message and count used in
    * {@link Greeter}.
    */
    static class DemoModule extends AbstractModule {
        @Provides
        @Count
        static Integer provideCount() {
        return 3;
        }

        @Provides
        @Message
        static String provideMessage() {
        return "hello world";
        }
    }

    static class Greeter {
        private final String message;
        private final int count;

        // Greeter declares that it needs a string message and an integer
        // representing the number of time the message to be printed.
        // The @Inject annotation marks this constructor as eligible to be used by
        // Guice.
        @Inject
        Greeter(@Message String message, @Count int count) {
        this.message = message;
        this.count = count;
        }

        void sayHello() {
        for (int i=0; i < count; i++) {
            System.out.println(message);
        }
        }
    }

    public static void main(String[] args) {
        /*
        * Guice.createInjector() takes one or more modules, and returns a new Injector
        * instance. Most applications will call this method exactly once, in their
        * main() method.
        */
        Injector injector = Guice.createInjector(new DemoModule());

        /*
        * Now that we've got the injector, we can build objects.
        */
        Greeter greeter = injector.getInstance(Greeter.class);

        // Prints "hello world" 3 times to the console.
        greeter.sayHello();
    }
}
```
&nbsp;&nbsp; `GuiceDemo`应用使用Guice创建了一个很小的对象依赖关系图，并用来创建Greeter实例。大型应用程序通常通过很多的`Module`对象来管理复杂的对象依赖关系。

## 下一章节
&nbsp;&nbsp;了解更多关于Guice的[核心设计](../mental-model/index.md)理念。