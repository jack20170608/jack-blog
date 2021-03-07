---
title: "Google guice系列(三)  心愿模型"
categories: ["Java", "guice"]
tags: ["guice","IOC","依赖注入"]
date: 2021-03-07T10:18:48+08:00
draft: false
toc: true
---

## Guice的心愿模型

> 学习关于`Key`,`Provider`并且为什么Guice只是一个Map。
  
&nbsp;&nbsp;当你阅读有关Guice的文章时，你经常会碰到这些流行的词，像“控制反转”，“好莱坞模式”，“注入”，这些可能会使你有点懵。但是在这些套话的下面的核心概念却并不复杂。实际上你可能已经自己实现了类似的东西，这篇文章就带你浏览一个最简单的模型在Guice中是如何实现的，这能帮助你更好的理解Guice是如何工作的。

> **Note**： 这篇文章假设你完全不了解依赖注入和Guice，它需要你有Java编程基础，包括Java的注解，方法引用以及Lambda表达式，包括基本的面向对象编程的思想和基本模式。

## Guice keys
&nbsp;&nbsp; Guice 使用Key来标识可以解析的依赖项。
&nbsp;&nbsp; 在[快速开始](../getting-started/index.md)文章的的`Greeter`类的构造方法中， 申明了2个依赖项，这2个依赖项在Guice内部拥有唯一的Key:
```java
@Message String ----> Key<String>
@Count int ----> Key<Integer>
```
&nbsp;&nbsp;`Key`最简单的形式就用Java里的类型来表示。
```java
// Identifies a dependency that is an instance of String.
Key<String> databaseKey = Key.get(String.class);
```
&nbsp;&nbsp;但是，一个应用程序经常会出现几个依赖项的类型相同，例如:
```java
final class MultilingualGreeter {
    private String englishGreeting;
    private String spanishGreeting;

    MultilingualGreeter(String englishGreeting, String spanishGreeting) {
        this.englishGreeting = englishGreeting;
        this.spanishGreeting = spanishGreeting;
    }
}
```
&nbsp;&nbsp;这种情况下，Guice使用[binding annotations](../binding-annotations/index.md)来区分类型相同的注入项，这就使得类型更加具体。
```java
final class MultilingualGreeter {
    private String englishGreeting;
    private String spanishGreeting;

    @Inject
    MultilingualGreeter(
        @English String englishGreeting, @Spanish String spanishGreeting) {
        this.englishGreeting = englishGreeting;
        this.spanishGreting = spanishGreeting;
    }
}
```
&nbsp;&nbsp;一个binding annotations的`Key`可以以下面这种方式创建: 
```java
Key<String> englishGreetingKey = Key.get(String.class, English.class);
Key<String> spanishGreetingKey = Key.get(String.class, Spanish.class);
```
&nbsp;&nbsp;当一个应用程序调用`injector.getInstance(MultilingualGreeter.class)`时，就会创建一个`MultilingualGreeter`的实例，类似下面的这段代码:
```java
// Guice internally does this for you so you don't have to wire up those
// dependencies manually.
String english = injector.getInstance(Key.get(String.class, English.class));
String spanish = injector.getInstance(Key.get(String.class, Spanish.class));
MultilingualGreeter greeter = new MultilingualGreeter(english, spanish);
```
&nbsp;&nbsp;总结以下： Guice `Key`是一个整合了可选的binding annotations的类型，用来唯一标识一个依赖项。

## Guice Provider 
&nbsp;&nbsp; Guice使用`Provider`来作为一个能够创建对象以满足依赖性需求的工厂。`Proider`是一个只有一个方法的接口:
```java
interface Provider<T> {
    /** Provide an instance of T **/
    T get();
}
```
&nbsp;&nbsp;每个直接实现了`Provider`接口的类，他们使用`Module`来配置Guice 注入器，然后Guice在内部自己为所有他能创建的对象创建`Provider`类。
&nbsp;&nbsp;下面这个例子的Guice module创建了2个`Provider`s:
```java
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
- Provider<String> 返回 "hello world" 字符串
- Provider<Integer> 会调用 provideCount 方法并返回整数 3

## Guice 的心愿模型
&nbsp;&nbsp;最简单就把Guice想象成一个大的map，它的类型可以描述为:`Map<Key<?>, Provider<?>>`。(当然，它不是类型安全的，因为这里面两个通配符的泛型)，这是一个最简单的模型，不是一个严格的实现。
&nbsp;&nbsp;像`Provider`s，大部分的应用都会使用`Module`s来配置`Key`到`Provider`的映射关系，而不会直接创建`Key`。

## 使用Modules来添加依赖关系
&nbsp;&nbsp; Module使用一堆配置逻辑代码来把东西添加到map中。目前有2个方式，一种是使用注解像`@Provider`，另外一种就是使用Guice DSL(Domain Specific Language)。

&nbsp;&nbsp;理论上，这些APIs只是提供了操纵Guice map的方法，而且这些APIs使用起来非常简单粗暴。下面有一些简单明了的例子，使用了Java8的语法。

| Guice DSL 语法                  | 模型表示                                        | 绑定方式                 |
| :------------------------------ | :---------------------------------------------- | :----------------------- |
| bind(key).toInstance(value)     | map.put(key, () -> value)                       | 实例绑定                 |
| bind(key).toProvider(provider)  | map.put(key, provider)                          | Provider 绑定            |
| bind(key).to(anotherKey)        | map.put(key, map.get(anotherKey))               | Linked 绑定              |
| @Provides Foo provideFoo(){...} | map.put(Key.get(Foo.class), module::provideFoo) | Provider 方法 binding) |

在上一章节中的`DemoModule` 向Guice map中添加了2个条目:

## 依赖注入

> 不要从Guice map以外的地方自己获取，而是在使用的地方申明

&nbsp;&nbsp;这是依赖注入的本质，如果你要需要申明东西，不要尝试从外面什么地方获取它，甚至请求一个类返回给你。相反你应该简单的申明说没有他们，我就无法工作，并且依赖别人给你你想要的东西。

&nbsp;&nbsp; 这个模型与大多数人对代码的理解相反，他更具申明性，而不是命令式的模型。这也就是为什么依赖注入经常被描述为控制反转的一种类型。

&nbsp;&nbsp;几种申明你所需要的依赖的方式如下: 

1. @Inject 构造函数参数输入
```java
class Foo {
    private Database database;

    @Inject
    Foo(Database database) {  // We need a database, from somewhere
        this.database = database;
    }
}
```

2. @Provides方法参数注入
```java
@Provides
Database provideDatabase(
    // We need the @DatabasePath String before we can construct a Database
    @DatabasePath String databasePath) {
  return new Database(databasePath);
}
```

## 依赖关系图
&nbsp; &nbsp;当注入一个对象，但是对象本身又依赖其他的对象，Guice会帮我们递归的注入依赖关系。你可以想象以下，为了注入一个`Foo`的对象，Guice像下面的代码创建了`Provider`的实现类。

```java
class FooProvider implements Provider<Foo> {
    @Override
    public Foo get() {
        Provider<Database> databaseProvider = guiceMap.get(Key.get(Database.class));
        Database database = databaseProvider.get();
        return new Foo(database);
        }
    }

class ProvideDatabaseProvider implements Provider<Database> {
    @Override
    public Database get() {
        Provider<String> databasePathProvider =
            guiceMap.get(Key.get(String.class, DatabasePath.class));
        String databasePath = databasePathProvider.get();
        return module.provideDatabase(databasePath);
    }
}
```
&nbsp;&nbsp;依赖关系行成了一个有向图，注入的工作原理就是从你需要的对象开始，对图进行深度优先遍历他的所有依赖项。

&nbsp;&nbsp;Guice Injector对象包含了整个依赖关系图。为了创建一个Injector，Guice需要验证整个关系图工作正常，不能出现悬空的情况，例如需要一个依赖项，但是该依赖项却没有Provider。如果Guice关系图由于任何原因导致无效，Guice就会抛出`CreationException`来描述错误的原因。

&nbsp;&nbsp;通常情况下，`CreationException`意味着你正在使用一些东西(像数据库)，但是你忘记把它包含在module里，或者是出现一些拼写错误。

## 下一章节
&nbsp;&nbsp; 学习如何使用[Scope](../scope/index.md)来管理Guice创建的对象的生命周期，并且包含添加对象依赖关系的更多的方式。



