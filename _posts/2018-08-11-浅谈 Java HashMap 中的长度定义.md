---
layout:     post
title:      浅谈 Java HashMap 中的长度定义
subtitle:   为何 Java HashMap 中的容量要定义为2的幂次方？
date:       2018-08-11
author:     fyypumpkin
header-img: img/home-bg-art.jpg
catalog: true
tags:
    - Java
    - HashMap
---

## 正文

### 我们先来看看HashMap初始化和扩容部分的源码吧

#### 看看 HashMap 的构造器

```java
    /**
     * Constructs an empty <tt>HashMap</tt> with the specified initial
     * capacity and load factor.
     *
     * @param  initialCapacity the initial capacity
     * @param  loadFactor      the load factor
     * @throws IllegalArgumentException if the initial capacity is negative
     *         or the load factor is nonpositive
     */
    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }
```

构造器中，调用了一个 tableSizeFor() 的函数，初始化的时候只会设置阈值，该函数会将传入的容量 initialCapacity 重新计算大于或者等于 initialCapacity 的最小的2的幂次方，
说的比较绕口，举个例子吧，看下面精简的代码：

```java
    ...
    tableSizeFor(15) ===> 会输出 16，16 = 2^4
    tableSizeFor(16) ===> 会输出 16，16 = 2^4
    tableSizeFor(17) ===> 会输出 32，32 = 2^5
    ...
```

从代码中可以看出，当入参为 15 的时候， 不满足 2 的幂次方，那么就将其设置为比15大的最小满足 2 的幂次方的数，也就是 16。
当入参为 16，即满足了幂次方的要求，那么返回的还是16， 当入参为 17 时候，不满足，那么返回 32。

回到上面的话题，可以看出，当创建一个 HashMap 的时候，如果指定了大小，那么就会将该大小进行一个 "标准化"，使其满足要求。

同时，HashMap 还包含了其扩容，相也可以知道，扩容一定是原容量进行左移一位来进行两倍扩容，这样可以保证容量还是2的幂次方，
下面我贴出了 HashMap 扩容的部分代码


```java
 final Node<K,V>[] resize() {
    ...
       int oldThr = threshold;
             int newCap, newThr = 0;
             if (oldCap > 0) {
                 if (oldCap >= MAXIMUM_CAPACITY) {
                     threshold = Integer.MAX_VALUE;
                     return oldTab;
                 }
                 else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                          oldCap >= DEFAULT_INITIAL_CAPACITY)
                     newThr = oldThr << 1; // double threshold
             }
    ...         
             Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    ...
    }
```

从代码的 `newCap = oldCap << 1` 可以验证我们刚才的猜想，同时阈值也会变为原来的两倍 `newThr = oldThr << 1`，（阈值初始化会根据负载因子来初始化，这里就不多阐述）同时会生成一个新容量的 Node 数组

说了这么多，还是没有说道，为何容量要设置成2的幂次方，我们来看看 HashMap 是如何将数据 put 的，

```java
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
            ...省略下面代码
```

我们不考虑 hash 碰撞了的情况和树化的情况，我只贴了一小部分的代码，从上面代码可以看到，HashMap 通过与运算来计算其放置的下标 `i = (n - 1) & hash`，一般我们利用 hash 码计算下标是会通过 hash吗对长度取模来获得，也就是 `h % length`, 
可能是这种计算的方式效率不够高，于是编写该源码的大师们采用了与运算来计算下标，大师们发现，当一个容量满足 2^n 时，`i = (n - 1) & hash` == `h % length` 的结果总是为true

对于 length = 16, 对应二进制 1 0000, length-1=0 1111 
假设此时 h = 17 . 
(1) 使用 h % length, 也就是 17 % 16, 结果是 1 
(2) 使用 h & (length - 1), 也就是 “1 0001 & 0 1111, 结果也是1 
我们会发现, 因为 0 1111 (length - 1) 低位都是 1, 进行 & 操作时, 就能成功保留 1 0001 对应的低位, 将高位的都丢弃, 低位是多少, 最后结果就是多少 
刚好低位的范围是 0 ~ 15, 刚好是长度为 length = 16 的所有索引

下面是我的测试代码

```java
public class Test {
    public static void main(String[] args) {
        System.out.println(Test.tableSizeFor(16));
        int h = "ha".hashCode();
        int length = 1 << 5; // 32
        int length2 = (1 << 5) - 1; // 31
        Test t = new Test();

        // 可以看到，当长度为2的幂次方时候，结果输出都为25
        System.out.println(t.getIndex(h, length) + " " + t.getIndexByPer(h, length));
        // 长度部位2的幂次方，结果就不同
        System.out.println(t.getIndex(h, length2) + " " + t.getIndexByPer(h, length2));

        // 结果
        // 25 25
        // 24 4

    }

    // length是数组的长度，h是hash码，计算对应下标，通过与运算
    private Integer getIndex (int h, int length) {
        return h & (length - 1);
    }

    // 通过取模
    private Integer getIndexByPer(int h, int length){
        return h % length;
    }

    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= 1000) ? 1000 : n + 1;
    }
}
```

> 本文首次发布于 [fyypumpkin Blog](http://fyypumpkin.github.io), 作者 [@fyypumpkin](http://github.com/fyypumpkin) ,转载请保留原文链接.> 