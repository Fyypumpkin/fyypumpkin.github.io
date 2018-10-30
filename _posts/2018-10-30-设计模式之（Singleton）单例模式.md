---
layout:     post
title:      设计模式之 （Singleton）单例模式
subtitle:   看了下 Singleton 单例模式，一个比较常用简单的模式设计
date:       2018-10-29
author:     fyypumpkin
header-img: img/post-bg-2015.jpg
catalog: true
tags:
    - Java
    - 设计模式
---

## Singleton 单例模式

单例模式相信大家都是耳熟能详的一个比较经典简单的设计模式，单例模式也会在很多地方碰到。

### 什么叫 Singleton 单例模式呢？

顾名思义，单例模式就是全局就只有单例的模式，也就是说，全局环境下，只会有一份对象，该对象会被复用，单例模式下，这个唯一的对象必须由单例类自己来创建，
并且单例类只能对外提供这一份创建好的对象，在单例模式下，首要解决的问题就是如何保证全局的唯一性，因为该模式比较简单，我直接通过不同的代码来表达实现单例模式
的不同方法。

### Singleton 单例模式的多种表达

> 1. EagerSingleton 饿汉式单例

所谓饿汉式，就是在类初始化阶段，就将这个唯一的实例创建完成，由于他的初始化是在类加载阶段完成，加载阶段会由虚拟机来保证线程的安全性，所以饿汉式的单例模式是线程安全的。

```java
public class test1 {
   // 饿汉式
   private static final test1 INSTANCE = new test1();
   
   private test1() {}
   
   public static test1 getInstance() {
       return INSTANCE;
   }
}
```

> 2. LazySingleton 懒汉式单例

懒汉式单例和饿汉式相对的，他的实例是在使用的时候才进行初始化，这就会带来线程安全性的问题，多个线程访问可能不能正确的保证单例，这就需要我们在代码中控制线程安全，
下面我会就线程安全和线程非安全来分析

> 2.1 LazySingleton 线程非安全模式

```java
class test2 {
    // 懒汉式
    private static test2 instance = null;

    private test2() { }

    public static test2 getInstance() {
        if (instance == null) {
            instance = new test2();
        }
        return instance;
    }
}
```

上面这段代码是线程非安全的，当一个线程访问 getInstance 的时候，进入 if 代码块，但还没有执行 new 操作，于是又有一个线程来访问，同样，由于上一个线程没有执行 new 操作，
所以也进入了 if 代码块，也进行了 new 操作，这样就失去了单例模式的意义，下面我会再贴一段线程安全的

> 2.2 LazySingleton 线程安全 version 1

```java
class test3 {
    // 懒汉式线程安全
    private static test3 instance = null;

    private test3() {
    }

    public static synchronized test3 getInstance() {
        if (instance == null) {
            instance = new test3();
        }
        return instance;
    }
}
```

简单粗暴，直接使用 synchronized 关键字，锁住整个方法，虽然保证了线程安全性，但是这段代码的效率会很低，因为当 instance 不为空的时候，依然要锁住 this 对象，下面贴出优化方案

> 2.3 LazySingleton 线程安全 version 2

```java
class test4 {
    // 懒汉式线程安全
    private static test4 instance = null;

    private test4() {
    }

    public static test4 getInstance() {
        if (instance == null) {
            synchronized (test4.class) {
                if(instance == null) {
                    instance = new test4();
                }
            }
        }
        return instance;
    }
}
```

可以看到，上面代码中使用了两个对于 instance 的判断，这也是为了防止有多个对象生成，加入一个线程进入第一个判断，随后又有一个线程进入，若没有第二层判断，那么第一个线程进入同步代码块，执行了 new 操作，
退出同步代码块后第二个线程再次进入同步代码块，则会再次进行 new 操作，用双重的判断就是为了防止进入同步代码块后的 instance 已经背上一个线程初始化过了。

可以看到，饿汉式单例模式的方式会在初始化的时候就生成好对象，在较小的系统中可能能够很好地运作，但在大型系统中，初始化过程会变得非常长，而且这些对象会占用系统资源，懒汉式虽然能够延迟加载，但是由于有锁，
实际运行效率肯定不会很高，下面再来介绍一种单例模式，也是目前主流的单例。

> 2.4 Initialization Demand Holder 线程安全 

```java
class test5 {
    // 内部类 Initialization Demand Holder (IoDH)
    private test5() {
        System.out.println("aa");
    }

    public static test5 getInstance() {
        return Holder.INSTANCE;
    }

    private static class Holder {
        private static final test5 INSTANCE = new test5();
    }
}
```

上面这段代码，在静态内部类中定义实例，外部直接返回内部类中定义的，由于类的初始化只会进行一次，切安全性由虚拟机保证，所以为线程安全的，这种方式有点像结合了饿汉式和懒汉式的优点而产生的。
最后在贴一种不常见的，但是确实安全性最高的，上面所有的单例在对象序列化反序列化的时候还是会生成新对象的，下面这种是也保证了序列化反序列化的安全性。

> 2.5 EnumSingleton 线程安全，序列化安全

```java
enum test6 {
    INSTANCE
}
```

哈哈，你没有看错，直接就是一个枚举就行，枚举默认构造器是private的，他的枚举值默认就 == public static final，不过这种方式的单例我还没有用过~

以上就是单例模式的全部信息啦~后续有新的会继续补充。


> 本文首次发布于 [fyypumpkin Blog](http://fyypumpkin.github.io), 作者 [@fyypumpkin](http://github.com/fyypumpkin) ,转载请保留原文链接.
