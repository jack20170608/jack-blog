---
title: "小白学编程之并发(十) 开发案例-Monitor"
categories: ["Java","小白学编程"]
tags: ["Java","并发编程","编程小白"]
date: 2020-11-11T17:24:42+08:00
draft: false
toc: true
---

&emsp;&emsp;这个案例来源于知乎上的一篇文章[《架构师写的BUG，非比寻常》](https://zhuanlan.zhihu.com/p/277529352)，感兴趣的朋友可以点开去看看。Java并发编程其实要理解透彻其实并不容易，但是如果有一些好的实际工程案例，带着问题来学可能事半功倍。

## 1. 需求
&emsp;&emsp;这个需求基本来源于这个文章，笔者这里稍微润色一下，就是公司的现有采用Spring Boot 全家的，一个已经上线的Rest API 后端程序。已经上线运行了一段时间，业务和运维发现有些时候某些API的响应时间比较长，业务部门要求IT定位问题并能做出相应的优化来提升业务体验。项目组长分析提出开发一个监控模块，能够统计每个API的请求次数、响应总时间以及平均响应时间，能发现有性能问题的接口。

&emsp;&emsp;系统当前API的接口数量为几百个左右，访问量50W量级每天，其实不算太多。这是一个并发量很高的需求，因为需要统计所有的接口，方法自然比单个接口的请求量都要大。因为使用了Spring Boot 全家桶，解决思路很简单，使用AOP把代码一包，每个接口从这里走一圈就搞定，于是第一版代码就出来了。

## 2. Monitor 0.1 版
&emsp;&emsp;由于大家都忙于紧急的业务需求开发，组长就把这个long term的需求就交给了刚刚毕业参加工作来公司的小A，让他来挑战一下，也顺便让他提升一下开发水平。小A很快就完成了开发，并提交了Pull Request。小A的代码如下。
```Java
package org.jack.monitor.impl;

import java.util.Map;
import java.util.StringJoiner;
import java.util.concurrent.ConcurrentHashMap;

public class MonitorImplV0 {
    public static class MonitorKey {
        public MonitorKey(String key, String desc){
            this.key = key;
            this.desc = desc;
        }
        private String key;
        private String desc;
    }

    public static class MonitorValue {
        int count ;
        double avgTime;
        Long totalTime ;

        @Override
        public String toString() {
            return new StringJoiner(", ", MonitorValue.class.getSimpleName() + "[", "]")
                    .add("count=" + count)
                    .add("avgTime=" + avgTime)
                    .add("totalTime=" + totalTime)
                    .toString();
        }
    }

    public void visit(String url, String desc, long timeCost){
        MonitorKey key = new MonitorKey(url, desc);
        MonitorValue value = monitors.get(key);
        if (null == value){
            value = new MonitorValue();
            monitors.put(key, value);
        }
        value.count++;
        value.totalTime += timeCost;
        value.avgTime = (double) value.totalTime / value.count;
    }

    public Map<MonitorKey, MonitorValue> getMonitors() {
        return monitors;
    }
    private Map<MonitorKey, MonitorValue> monitors = new HashMap<>();
}
```
&emsp;&emsp;小B来公司有一段时间，对多线程有一些了解，看了看代码，直接Reject说代码不满足多线程环境下的应用条件，并留了一些修改建议。
1. HashMap 线程不安全，多线程环境下必须使用ConcurrentHashMap
2. MonitorValue 里面应该使用原子类

&emsp;&emsp;小B比较热心，写了一个单元测试告诉小A，ConcurrentHashMap以及JAVA原子类的使用，只有这样才能保证统计结果的准确性。
```java
    @Test
    public void testHashMap() throws Throwable{
        Map<String, Integer> map = new HashMap<>();
        String key = "key";
        Runnable runnable = () -> {
            for(int i = 0; i < 1000; i++) {
                Integer count = map.get(key);
                if (null == count) {
                    map.put(key, 1);
                } else {
                    map.put(key, ++count);
                }
            }
        };

        Thread t1 = new Thread(runnable);
        Thread t2 = new Thread(runnable);

        t1.start();
        t2.start();
        t1.join();
        t2.join();

        System.out.println(map.get(key));
    }

    @Test
    public void testCurrentHashMap() throws Throwable{
        Map<String, AtomicInteger> map = new ConcurrentHashMap<>();
        String key = "key";
        Runnable runnable = () -> {
            for(int i = 0; i < 1000; i++) {
                AtomicInteger count = map.get(key);
                if (null == count) {
                    map.put(key, new AtomicInteger(1));
                } else {
                    count.incrementAndGet();
                    map.put(key, count);
                }
            }
        };

        Thread t1 = new Thread(runnable);
        Thread t2 = new Thread(runnable);
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        System.out.println(map.get(key));
    }
```

## 3.  Monitor 1.0 版
&emsp;&emsp; 小A仔细的琢磨了下单元测试，也好好的学了下Java多线程相关的入门资料，于是1.0版很快就出来了，并重新提交了pull request给小B做code review 。
```java
package org.jack.monitor.impl;

import java.util.Map;
import java.util.StringJoiner;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.AtomicLong;


public class MonitorImplV1 {

    public static class MonitorKey {

        public MonitorKey(String key, String desc){
            this.key = key;
            this.desc = desc;
        }
        private String key;

        private String desc;
    }

    public static class MonitorValue {
        AtomicInteger count = new AtomicInteger();
        double avgTime;
        AtomicLong totalTime = new AtomicLong();

        @Override
        public String toString() {
            return new StringJoiner(", ", MonitorValue.class.getSimpleName() + "[", "]")
                    .add("count=" + count)
                    .add("avgTime=" + avgTime)
                    .add("totalTime=" + totalTime)
                    .toString();
        }
    }
    public void visit(String url, String desc, long timeCost){
        MonitorKey key = new MonitorKey(url, desc);
        MonitorValue value = monitors.get(key);
        if (null == value){
            value = new MonitorValue();
            monitors.put(key, value);
        }
        value.count.getAndIncrement();
        value.totalTime.getAndAdd(timeCost);
        value.avgTime = (double) value.totalTime.get() / value.count.get();
    }
    public Map<MonitorKey, MonitorValue> getMonitors() {
        return monitors;
    }
    private Map<MonitorKey, MonitorValue> monitors = new ConcurrentHashMap<>();
}
```
&emsp;&emsp; 这次小B很爽快的approve了。小A激动的赶紧在SIT和UAT上完成测试，一切看起来挺顺利的，1周后，顺利的上了生产环境，小A完成了入职以来的第一个task。可是好景不长，1个月后，运维报告这个API接口，JVM内存经常报警，GC次数越来越高而且内存使用越来越高，估计会产生OOM的风险。

&emsp;&emsp; 项目组长这次坐不住了，拉了一个Java经验丰富的老鸟小Y一起来查找内存泄漏原因，小Y三下五除二，先沟通运维在业务量少的时候dump了生产环境线程栈和内存堆。结果发现内存堆中```MonitorKey```数量居然超过了1000W个，这个明显不正常，然后查看代码使用，发现MonitorKey的问题，由于忘了给MonitorKey重写equals和hashCode方法，导致MonitorKey使用的是Object的默认方法，而Object的默认方法equal是比较对象引用，导致每次调用visit方法都会往内存写一个新对象。找到原因后，小A重新写了2.0版，而且小Y提醒小A任何代码改动一定要有单元测试。

&emsp;&emsp; 例会时，项目组长同时又把单元测试重要性传达了一遍。

## 4. Monitor 2.0 版

小A吸取了上次的教训，把单元测试看的很重，2.0版还包括了一个很Nice的单元测试。

```java
package org.jack.monitor.impl;

import com.google.common.base.Objects;

import java.util.Map;
import java.util.StringJoiner;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.AtomicLong;


public class MonitorImplV2 {

    public static class MonitorKey {

        public MonitorKey(String key, String desc) {
            this.key = key;
            this.desc = desc;
        }

        private String key;

        private String desc;

        @Override
        public boolean equals(Object o) {
            if (this == o) return true;
            if (o == null || getClass() != o.getClass()) return false;
            MonitorKey that = (MonitorKey) o;
            return Objects.equal(key, that.key) &&
                    Objects.equal(desc, that.desc);
        }

        @Override
        public int hashCode() {
            return Objects.hashCode(key, desc);
        }
    }

    public static class MonitorValue {
        AtomicInteger count = new AtomicInteger(0);
        float avgTime;
        AtomicLong totalTime = new AtomicLong(0);

        @Override
        public String toString() {
            return new StringJoiner(", ", MonitorValue.class.getSimpleName() + "[", "]")
                    .add("count=" + count)
                    .add("avgTime=" + avgTime)
                    .add("totalTime=" + totalTime)
                    .toString();
        }
    }
    public void visit(String url, String desc, long timeCost) {
        MonitorKey key = new MonitorKey(url, desc);
        MonitorValue value = monitors.get(key);
        if (null == value) {
            value = new MonitorValue();
            /* by xiaoY 
            mock some internal CPU delay
            try {
                 Thread.sleep(100L);
             } catch (Exception ignore) {}
             */
            monitors.put(key, value);
            // System.out.println(Thread.currentThread().getName() + " lucky guy....");
        // } else {
            // System.out.println(Thread.currentThread().getName());
        }
        value.count.getAndIncrement();
        value.totalTime.getAndAdd(timeCost);
        value.avgTime = value.totalTime.get() / value.count.get();
    }
    public Map<MonitorKey, MonitorValue> getMonitors() {
        return monitors;
    }

    private Map<MonitorKey, MonitorValue> monitors = new ConcurrentHashMap<>();
}

```
&emsp;&emsp;小A在单元测试里模拟了3000个线程，轮流call 3个API的情景，每个api设定响应时间为10，最后验证monitors valueSet对象的数量为3个，并且每个对象的count值为1000，平均响应时长为10。
```java
package org.jack.monitor;

import com.google.common.base.Stopwatch;
import org.jack.monitor.impl.*;
import org.junit.Assert;
import org.junit.Test;

import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

public class MonitorTest {

    static Map<Integer, String> API = new HashMap<>();
    static int THREAD_NUM_1000 = 1000;
    static int THREAD_NUM_3000 = 3000;

    static {
        API.put(1, "http://localhost:8080/account/getById/100");
        API.put(2, "http://localhost:8080/account/getAll");
        API.put(3, "http://localhost:8080/account/getByName&name=jack");
        API.put(4, "http://localhost:8080/account/deleteById/100");
    }

    @Test
    public void testMonitorV2() throws Throwable{
        MonitorImplV2 monitorV2 = new MonitorImplV2();
        Thread[] threads = new Thread[THREAD_NUM_3000];
        Stopwatch stopwatch = Stopwatch.createStarted();
        for(int i = 0; i < THREAD_NUM_3000; i++){
            int mod3 =  i % 3;
            Thread t = new Thread(() -> monitorV2.visit(API.get(mod3), null, 10L));
            threads[i] = t;
            t.start();
        }
        //Wait the thread finish
        for (Thread thread : threads) {
            thread.join();
        }
        System.out.println("TimeCost: " + stopwatch.elapsed(TimeUnit.MILLISECONDS) + "ms");
        stopwatch.stop();
        Assert.assertEquals(3, monitorV2.getMonitors().size());
        monitorV2.getMonitors().values().forEach(v -> {
            Assert.assertEquals("MonitorValue[count=1000, avgTime=10.0, totalTime=10000]" , v.toString());
        });
    }
}
```

```text
TimeCost: 1661ms
Test Pass!!!
```
&emsp;&emsp;本地的单元测试的结果很OK，看这里打印出来的结果，挺完美，代码提交，似乎该结束这个该死的task了。
可是不久后，项目组的其他成员经常抱怨，项目的jenkins自动化流水线有时会因为小A的这个单元测试而失败，但是重新build之后，又可以成功。小A心里有点犯嘀咕，what a fuck，还有这样的事。这次小A比较低调，虚心向小Y请教这件事情。小Y毕竟经验丰富，发现小A在使用ConcurrentHashMap时存在问题，而且这个问题有时单元测试都不一定能发现因为CPU实在是太快了。

&emsp;&emsp;问题就在visit方法，它首先拿出了key，然后判空，再塞值，这明显不是一个原子操作。
```Java
        MonitorValue value = monitors.get(key);
        if (null == value) {
            value = new MonitorValue();
            monitors.put(key, value);        
        }
```
如果两个线程按照下面的步骤来执行。
```text 
线程1：获取key为a的值
线程2：获取key为a的值
线程1：a为null，生成一个b
线程2：a为null，生成一个c
线程1：保存a=b
线程2：保存a=c
```
此时，B丢了。

小Y告诉小A，重现这个问题其实也很简单，在判空的if里面人为放个陷阱来模拟CPU内部的某种延迟(因为CPU的延迟我们无法预知)，代码修改如下：
```Java
        MonitorValue value = monitors.get(key);
        if (null == value) {
            value = new MonitorValue();
            // by xiaoY mock some internal CPU delay            
            try {
                 Thread.sleep(100L);
             } catch (Exception ignore) {}
            monitors.put(key, value);
            System.out.println(Thread.currentThread().getName() + " lucky guy....");
         } else {
             System.out.println(Thread.currentThread().getName());
        }
```
重新运行单元测试，发现单元测试过不了,有一些数据都丢掉了。小A这下明白了，然后小Y说最直接的解决办法就是在visit方法上加synchronized关键字，简单粗暴、当然不适应于性能要求非常苛刻的场景。

```text 
org.junit.ComparisonFailure: 
Expected :MonitorValue[count=1000, avgTime=10.0, totalTime=10000]
Actual   :MonitorValue[count=819, avgTime=10.0, totalTime=8190]
```

于是小A，很快的发布了3.0版，终于解决了这个问题。小A一直在思考小Y说的话，如果是性能要求很高的情况下该怎么做呢？