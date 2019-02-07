---
layout:     post
title:      Java LinkedList 源码分析
subtitle:   了解 LinkedList 源码及其工作原理
date:       2019-02-06
author:     fyypumpkin
header-img: img/post-bg-debug.png
catalog: true
tags:
    - Java
    - LinkedList
    - Java 集合
---

## 正文

上一篇博客看了下 ArrayList, 这次来看一下 LinkedList，集合中，LinkedList 是一个比较重要的常用集合，LinkedList 实现了 List 接口，也实现了 Deque 接口，所以 LinkedList 既可以作为正常的 list 类使用，
也可以作为双向队列使用，同时还能作为栈使用。

LinkedList 内部实现是一个双向的链表，所以在初始化 LinkedList 的时候，不需要指定容量，因此其构造器也很简单。

```java

   //    todo 双向链表
    private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
    
    public LinkedList() {
    }
    
    public LinkedList(Collection<? extends E> c) {
        this();
        addAll(c);
    }
  public boolean addAll(Collection<? extends E> c) {
  // size 初始值是 0, 表示从 size 位置开始赋值进去
        return addAll(size, c);
    }

    public boolean addAll(int index, Collection<? extends E> c) {
    // 检查下标
        checkPositionIndex(index);

        Object[] a = c.toArray();
        int numNew = a.length;
        // 如果入参集合是空的，就返回
        if (numNew == 0)
            return false;

        Node<E> pred, succ;
//        todo 寻找 index 位置和 index 前驱节点
        if (index == size) {
            succ = null;
            pred = last;
        } else {
            succ = node(index);
            pred = succ.prev;
        }

        for (Object o : a) {
            @SuppressWarnings("unchecked") E e = (E) o;
            Node<E> newNode = new Node<>(pred, e, null);
            if (pred == null)
                first = newNode;
            else
                pred.next = newNode;
            pred = newNode;
        }

// succ == null 说明就是从最后一个元素开始赋值，赋值完毕，将 last 设置为当前的 pred
        if (succ == null) {
            last = pred;
        } else {
        // 否则说明是在链表中间开始赋值，赋值完毕后， last 不需要更改，但是赋值位置的顺序需要变更
            pred.next = succ;
            succ.prev = pred;
        }

        size += numNew;
        modCount++;
        return true;
    }
```

可以看到，LinkedList 中的 Node 节点就是一个双向的链表，
构造器一共有两种，一种是空构造，一种是复制的构造，将其他集合类型的数据复制到新的 LinkedList 中，通过 addAll 方法，addAll 方法也比较简单。

看一下内部私有的一些操作方法

linkFirst 方法

```java
    /**
     * Links e as first element.
     * todo 链到链头上
     */
    private void linkFirst(E e) {
        final Node<E> f = first;
        final Node<E> newNode = new Node<>(null, e, f);
        // 链表头设置为当前 node
        first = newNode;
        //        todo 如果当前是空的链表，就将链尾设置为当前的 node，否则，就将之前的头结点的前驱设置为当前节点 
        if (f == null)
            last = newNode;
        else
            f.prev = newNode;
        size++;
        modCount++;
    }
```

linkLast 方法

```java
   /**
     * Links e as last element.
     * todo 链到链尾上
     */
    void linkLast(E e) {
        final Node<E> l = last;
        final Node<E> newNode = new Node<>(l, e, null);
//        todo 将 last 指针指向当前节点
        last = newNode;
//        todo 如果之前的 last 是空的，就将头结点设置当前节点，否则将之前的尾节点的后继设置为当前节点
        if (l == null)
            first = newNode;
        else
            l.next = newNode;
        size++;
        modCount++;
    }
```

linkBefore 方法

```java
   /**
     * Inserts element e before non-null Node succ.
     * todo 链接到 succ 的前驱上
     */
    void linkBefore(E e, Node<E> succ) {
        // assert succ != null;
        final Node<E> pred = succ.prev;
//        todo 将 succ 的前驱设置为新节点前驱， succ 设置为新节点后继
        final Node<E> newNode = new Node<>(pred, e, succ);
        succ.prev = newNode;
        if (pred == null)
            first = newNode;
        else
            pred.next = newNode;
        size++;
        modCount++;
    }
```

unlinkFirst 方法

```java
    /**
     * Unlinks non-null first node f.
     * todo 删除链头
     */
    private E unlinkFirst(Node<E> f) {
        // todo f 是头结点
        // assert f == first && f != null;
        final E element = f.item;
        final Node<E> next = f.next;
        f.item = null;
        f.next = null; // help GC
        first = next;
//        todo 如果头结点的后继是 null，说明已经是最后一个元素，那么就将 last 设置为 null，否则就将 next 的前驱设置为 null，成为新的头结点
        if (next == null)
            last = null;
        else
            next.prev = null;
        size--;
        modCount++;
        return element;
    }
```

unlinkLast 方法

```java
    /**
     * Unlinks non-null last node l.
     * todo 删除链尾
     */
    private E unlinkLast(Node<E> l) {
//        todo l 是尾节点并且 l 不等于 null
        // assert l == last && l != null;
        final E element = l.item;
        final Node<E> prev = l.prev;
        l.item = null;
        l.prev = null; // help GC
        last = prev;
//        todo 如果尾节点的前驱是 null，说明尾节点 == 头结点，那么就将头结点指针也设置为 null，否则就将前驱节点的后继设置为 null
        if (prev == null)
            first = null;
        else
            prev.next = null;
        size--;
        modCount++;
        return element;
    }
```

unlink 方法

```java
   /**
     * Unlinks non-null node x.
     * todo 删除指定节点，这里的两个操作分别对前驱节点和后继节点操作，单独处理前驱节点和后继节点以及头尾指针的关系
     */
    E unlink(Node<E> x) {
        // assert x != null;
        final E element = x.item;
        final Node<E> next = x.next;
        final Node<E> prev = x.prev;
//      todo 如果前驱节点 == null，说明是头结点，就将头结点设置为 next，否则就将 x 节点的前驱节点的后继节点设置为 x 的后继节点
        if (prev == null) {
            first = next;
        } else {
            prev.next = next;
            x.prev = null;
        }
//      todo 如果 next 节点是空的，就说明 next 节点是原先的尾节点，就需要将尾节点指针 last 重新赋值，否则就将 next 节点的前驱设置为 x 节点的前驱
        if (next == null) {
            last = prev;
        } else {
            next.prev = prev;
            x.next = null;
        }
//      todo 将 x 节点的内容设置为 null，方便 gc
        x.item = null;
        size--;
        modCount++;
        return element;
    }
```

node 方法

```java
    /**
     * Returns the (non-null) Node at the specified element index.
     * todo 寻找指定下标节点
     */
    Node<E> node(int index) {
        // assert isElementIndex(index);
//      todo 如果节点小于 size 的一半，从头开始寻找，否则从尾部开始寻找
        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
```

LinkedList 的公用方法大部分都是基于上面的这些内部方法实现的，上面的方法包含了几乎所有对 LinkedList 的操作，有兴趣的可以看看公用的方法，实际上就是调用上面的这些方法实现的，常用的 get/set 类的方法就不展开说了，
下面说一下除了上面以及常用的 get/set 方法以为的方法

toArray 方法

```java
    public Object[] toArray() {
        Object[] result = new Object[size];
        int i = 0;
        for (Node<E> x = first; x != null; x = x.next)
            result[i++] = x.item;
        return result;
    }
    
    public <T> T[] toArray(T[] a) {
//        todo 如果传入的数组容量不够，会通过反射新建一个
        if (a.length < size)
            a = (T[]) java.lang.reflect.Array.newInstance(
                    a.getClass().getComponentType(), size);
        int i = 0;
        Object[] result = a;
        for (Node<E> x = first; x != null; x = x.next)
            result[i++] = x.item;

        if (a.length > size)
            a[size] = null;

        return a;
    }
```

特别小心第二个 toArray 方法，是可以带外部数组的，如果带进来的数组容量够的话，就会使用，不够的情况会新建一个，所以一定要使用返回值，不要忽略返回值

indexOf 方法 （lastIndexOf 方法只是遍历的顺序不同而已）

```java
    public int indexOf(Object o) {
        int index = 0;
        //todo 遍历找到第一个符合条件的元素，从头开始，找不到返回 -1
        if (o == null) {
            for (Node<E> x = first; x != null; x = x.next) {
                if (x.item == null)
                    return index;
                index++;
            }
        } else {
            for (Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item))
                    return index;
                index++;
            }
        }
        return -1;
    }
```


> 本文首次发布于 [fyypumpkin Blog](http://fyypumpkin.github.io), 作者 [@fyypumpkin](http://github.com/fyypumpkin) ,转载请保留原文链接.