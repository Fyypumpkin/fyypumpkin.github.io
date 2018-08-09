---
layout:     post
title:      Java intern() 方法解析
subtitle:   Java String.intern()方法，字符串常量池解析
date:       2018-08-09
author:     fyypumpkin
header-img: img/post-bg-alibaba.jpg
catalog: true
tags:
    - Java
    - 常量池
---

<p style="color: red">注: 本文中所在的环境为jdk1.8的版本，1.6版本以及之前版本的本文中所测试的代码结果是不同的，本文之探讨1.8以及之后版本的jdk环境下的测试</p>

## **String.intern()**原理浅析

String.intern() 是一个 native 的方法，也就是通过 C++ 来实现的，底层调用的是 C++ 的 `StringTable::intern` 方法。

```java
    /**
     * Returns a canonical representation for the string object.
     * <p>
     * A pool of strings, initially empty, is maintained privately by the
     * class {@code String}.
     * <p>
     * When the intern method is invoked, if the pool already contains a
     * string equal to this {@code String} object as determined by
     * the {@link #equals(Object)} method, then the string from the pool is
     * returned. Otherwise, this {@code String} object is added to the
     * pool and a reference to this {@code String} object is returned.
     * <p>
     * It follows that for any two strings {@code s} and {@code t},
     * {@code s.intern() == t.intern()} is {@code true}
     * if and only if {@code s.equals(t)} is {@code true}.
     * <p>
     * All literal strings and string-valued constant expressions are
     * interned. String literals are defined in section 3.10.5 of the
     * <cite>The Java&trade; Language Specification</cite>.
     *
     * @return  a string that has the same contents as this string, but is
     *          guaranteed to be from a pool of unique strings.
     */
    public native String intern();
```

上面的代码块是 java 中的 intern 方法，可以看到，源码中解释，当调用 intern 方法的时候，
如果常量池如果已经包含了这个字符串，则直接返回池中的字符串，如果字符串在池中不存在，则先将该字符串添加到
池子中，然后返回该对象的引用 （说明一下，7之后的版本常量池移入堆中，常量池可以存放堆中对象的引用，这也是和6以及之前版本最大的不同，也是导致运行结果不同的原因）

上一个例子说明下：
我直接使用了网上的例子，有兴趣的可以去看一下[原文](https://tech.meituan.com/in_depth_understanding_string_intern.html)

```java
public static void main(String[] args) {
    String s = new String("1");
    s.intern();
    String s2 = "1";
    System.out.println(s == s2);

    String s3 = new String("1") + new String("1");
    s3.intern();
    String s4 = "11";
    System.out.println(s3 == s4);
}
```

打印的结果是 `false true`

```java
public static void main(String[] args) {
    String s = new String("1");
    String s2 = "1";
    s.intern();
    System.out.println(s == s2);

    String s3 = new String("1") + new String("1");
    String s4 = "11";
    s3.intern();
    System.out.println(s3 == s4);
}
```

打印的结果是 `false false`

下面来分析一下，为什么会产生这样的结果

![](https://tech.meituan.com/img/in_depth_understanding_string_intern/jdk7_1.png)

先来看第一段代码，先看 `s` 和 `s2` ，`s` 直接通过 new String("1") 来生成，实际上 new String("1") 的过程会创建两个对象，
一个是实例对象，一个常量池对象，分别为 new String() 和 "1"，其中，我的理解是堆中的实例对象指向了常量池中的对象，但是`s`拿的是
实例对象的引用，再通过 s.intern() 发现常量池中已经有相应的值了，就会返回常量池中的值，这时候，在产生一个 `s2`，直接使用字面量，
则会直接从常量池中拿，所以 `s` 的引用是堆中的实例对象的，而 `s2` 的引用是直接指向常量池的，显然两个不是同一个引用，所以返回 `false`
再看 `s3` 和 `s4`，当有字符串相加的操作是，底层会使用 StringBuilder 来进行 append 操作，最终的值是不会进入常量池的，所以 `s3` 的引用是
堆中对象的引用，这时候调用 s3.intern() 方法，发现常量池中并没有相应的值，于是就会直接将堆中对象的引用保存（ jdk7 与 jdk6 的不同， jdk6会在生成一份），
这时候在生成一个对象 `s4` ，发现常量池中已经有相应的值，就直接获取，这是常量池中的值其实就是堆对象中的引用，和 `s3` 属于同一个，所以最终就返回了 `true`

![](https://tech.meituan.com/img/in_depth_understanding_string_intern/jdk7_2.png)

再来看一下第二段代码，第二段代码就是调用 intern 的顺序和第一段代码稍有不同，当然结果也是截然不同的，对于 `s` 和 `s2` 来说，实际上 intern 的顺序并不会影响这两个比较的结果，
因为调用 intern 前，实际上常量池中已经生成了相应的值，主要分析一下 `s3` 和 `s4`，实际上这个时候的 `s3` 和 `s4` 的情况就类似于 `s` 和 `s2` 了，
先通过 new 字段创建一个 `s3` ，那么堆中就会生成相应的实例，但由于没有调用 intern 方法，则常量池不会有相应的值或者引用，再通过字面量创建一个 `s4`，这时候就会直接去查找常量池中是否有相应的值，
发现常量池中并没有 "11" 这个值，那么就会在常量池中生成一份，并将该值的引用返回给 `s4` ，在调用 intern 方法，则只是会返回常量池中的值，并不会于堆中的对象建立引用关系，所以显然会返回 `false`

上述大致讲了一下为何 intern 会导致不同的结果，下面我贴出自己写的代码供大家参考一下，代码里包含了大量的注解来解释原因，当然可能有不准确的地方，如果大家觉得有问题，也可以给我反馈一下，共同学习

```java
public static void main(String[] args) {
        // new出来的直接会指向堆中的实例对象。同时发现字符串常量池中没有响应的字面量，于是会在字符串常量池中新建一个字面量为"hello"的常量，同时方法区中的会存放一份引用，下次在创建时，会直接返回该引用
        String str1 = new String("hello");
        // 直接使用字面量创建字符串，会返回常量池中的引用
        String str2 = "hello";

        // 会返回false，原因是str1指向的是堆中新建的实例，而str2中指向的是字符创常量池中的字面量，两者的引用不同
        System.out.println(str1 == str2);

        // 会返回false，因为str1指向的是堆中的，str intern指向的是常量池中的
        System.out.println(str1.intern() == str1);

        // 会返回true，原因是str1.intern方法会将该值加入到常量池中，并且返回常量池的引用，但是由于常量池中已经存在，所以intern返回的引用和str2的引用是同一个
        System.out.println(str1.intern() == str2);

        // String的加法操作会转换成StringBuilder的append操作，最后会使用toString方法返回一个字符串对象，
        // 由于该字符串在常量池中已经存在了，所以并不会触发赋值操作，导致常量池中的引用并不指向堆中的，str3指向堆中的对象。str3.intern指向常量池中的对象
        String str3 = new String("he") + new String("llo");

//         String str4 = new StringBuffer("he").append("llo").toString(); // 和str3等同

        String str5 = new String("a") + new String("b");
        // 返回true，验证一下上面的说法，调用intern后，发现常量池中没有响应的"ab"，所以直接在常量池中讲str5这个的对象引用进行保存，并返回接受，所以当常量池中没有相关数据的时候，
        // 就会在常量池中创建，常量池中会保存堆中对象的引用，（而new String则会在常量池中新增一份，所以new String拿到的并不是同一个）
        System.out.println(str5.intern() == str5);

        String str6 = new StringBuilder("1").append("111").toString();
        // 返回true，原因是Stringbuilder("1").append("111").toString();
        // 等价于 String a = "1"; String b = "111" StringBuilder(a).append(b).toString()，
        // 可以得知，1111是在堆中创建的，于是intern就会返回堆中的地址放入常量池
        System.out.println(str6.intern() == str6);

        String str7 = new StringBuilder("2").toString();
        // 返回false，原因是Stringbuilder("1")toString();
        // 等价于 String a = "2" StringBuilder(a).toString()，
        // 可以得知，2是在常量池中创建的，于是intern就会直接返回常量池中的地址，而new出来的地址会在堆中，所以是false，同样适用于new Sting（）
        System.out.println(str7.intern() == str6);

    }
```

上述的代码在我的 [Github](https://github.com/Fyypumpkin/blog-demo/tree/master/src/main/java/cn/fyypumpkin/blog/constant) 中都上传了

感谢美团技术团队分享的 [深入解析String#intern](https://tech.meituan.com/in_depth_understanding_string_intern.html) 文章，本文中有很多来自于这里

> 本文首次发布于 [fyypumpkin Blog](http://fyypumpkin.github.io), 作者 [@fyypumpkin](http://github.com/fyypumpkin) ,转载请保留原文链接.