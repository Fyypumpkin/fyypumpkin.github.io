---
layout:     post
title:      Java Spliterator 以及 Iterator 分析，基于 ArrayList
subtitle:   了解 ArrayList 中的 ArrayListSpliterator 和 Iterator
date:       2019-02-08
author:     fyypumpkin
header-img: img/post-bg-debug.png
catalog: true
tags:
    - Java
    - Spliterator
    - Java 集合
---

## 正文

我们在使用集合的时候，经常会使用到集合中的迭代器，今天我们就来讲讲集合中的迭代器。

在 ArrayList 源码中，有一个方法叫做 iterator，这个方法会给我们返回一个迭代器，我们就可以使用这个迭代器对集合中的元素进行访问。

```java
{
    List<String> list = new ArrayList();
    list.add("one");
    list.iterator().next();
}
```

进入到 iterator 方法中，很简单的一行代码

```java
  public Iterator<E> iterator() {
        return new Itr();
    }
```

其中 Itr 是一个内部类，我在这个类中加了注释，这个类就是集合中迭代器最重要的部分

```java
   private class Itr implements Iterator<E> {
//        下一次要访问的指针
        int cursor;       // index of next element to return
//        最后被访问的元素下标（在 remove 的时候，必须先被访问过才能 remove）
        int lastRet = -1; // index of last element returned; -1 if no such
//        modCount，在使用迭代器过程中，该值的改变会导致异常
        int expectedModCount = modCount;

        public boolean hasNext() {
            return cursor != size;
        }

        @SuppressWarnings("unchecked")
        public E next() {
            checkForComodification();
            int i = cursor;
            if (i >= size)
                throw new NoSuchElementException();
//            引用外部的 elementData， elementData 不是 volatile 的
            Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length)
                throw new ConcurrentModificationException();
            cursor = i + 1;
            return (E) elementData[lastRet = i];
        }

        // 在迭代器中要使用迭代器中的 remove 方法
        public void remove() {
            if (lastRet < 0)
                throw new IllegalStateException();
            checkForComodification();

            try {
//                通过持有的外部引用调用外部的 remove 方法，移除后，继续讲 lastRet 设置为 -1
                ArrayList.this.remove(lastRet);
                cursor = lastRet;
                lastRet = -1;
                expectedModCount = modCount;
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }

        @Override
        @SuppressWarnings("unchecked")
        public void forEachRemaining(Consumer<? super E> consumer) {
//            这是一个遍历的方法，会从 cursor 位置开始不断调用 consumer
            Objects.requireNonNull(consumer);
            final int size = ArrayList.this.size;
            int i = cursor;
            if (i >= size) {
                return;
            }
            final Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length) {
                throw new ConcurrentModificationException();
            }
            while (i != size && modCount == expectedModCount) {
                consumer.accept((E) elementData[i++]);
            }
            // update once at end of iteration to reduce heap write traffic
            cursor = i;
            lastRet = i - 1;
            checkForComodification();
        }

        final void checkForComodification() {
            // 这里会检查 遍历中的 modCount 和期望的是否一致，不一致会抛出异常
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
    }
```

迭代器并不难，代码量也很小，上面的注释已经很详细了，就不再过多的解释了，下面来看一下 java 1.8 最新加入的分割迭代 Spliterator，怎么理解这个呢，我上一段代码

```java
    public static void main(String[] args) {
        List<String> list = new ArrayList<>();
        list.add("1");
        list.add("2");
        list.add("3");
        list.add("4");
        list.add("5");
        list.add("6");
        list.add("7");
        list.add("8");
        list.add("9");

        Spliterator<String> spliterator = list.spliterator();

        Spliterator<String> stringSpliterator = spliterator.trySplit();

        spliterator.forEachRemaining(System.out::println);
        
        System.out.println();

        stringSpliterator.forEachRemaining(System.out::println);
    }
```

看一下输出

```java
5
6
7
8
9

1
2
3
4
```

可以看到，通过 Spliterator 将数据分割成多分，这样的好处就是可以并行的去处理集合中的数据，看一下 ArrayList 中的源码是怎么样的

```java
static final class ArrayListSpliterator<E> implements Spliterator<E> {

        private final ArrayList<E> list;
        // todo 起始下标
        private int index; // current index, modified on advance/split
//        todo 结束下标 （如果值小于 0，则会在使用的时候，用 list 的 size 来填充）
        private int fence; // -1 until used; then one past last index
        private int expectedModCount; // initialized when fence set

        /** Create new spliterator covering the given  range */
        ArrayListSpliterator(ArrayList<E> list, int origin, int fence,
                             int expectedModCount) {
            this.list = list; // OK if null unless traversed
            this.index = origin;
            this.fence = fence;
            this.expectedModCount = expectedModCount;
        }

        private int getFence() { // initialize fence to size on first use
            int hi; // (a specialized variant appears in method forEach)
            ArrayList<E> lst;
//            todo 初始化 fence ，-1 的时候会使用 list 的 size 来填充
            if ((hi = fence) < 0) {
//                todo 当前 fence 小于 0， 如果 list 是空的，返回 0，否则返回 list 的 size
                if ((lst = list) == null)
                    hi = fence = 0;
                else {
                    expectedModCount = lst.modCount;
                    hi = fence = lst.size;
                }
            }
            return hi;
        }

        public ArrayListSpliterator<E> trySplit() {
//            todo hi == fence， lo == index， mid = （lo + hi） / 2
            int hi = getFence(), lo = index, mid = (lo + hi) >>> 1;
//            todo 如果 lo >= mid，说明待拆分的 list 太小了，返回 null （这里要注意空指针）,这里可以看出来，他是向前拆分的，怎么说呢，我画一个图
//            todo split1 = 1 2 3 4 5 6 7 8 9
//            todo split2 = split1.trySplit()
//            todo split1 = 5 6 7 8 9 [4 - 9)
//            todo aplit2 = 1 2 3 4 [0 - 4)
            return (lo >= mid) ? null : // divide range in half unless too small
                new ArrayListSpliterator<E>(list, lo, index = mid,
                                            expectedModCount);
        }

        public boolean tryAdvance(Consumer<? super E> action) {
            if (action == null)
                throw new NullPointerException();
            int hi = getFence(), i = index;
            if (i < hi) {
//                todo 起始下标 +1
                index = i + 1;
//                todo 返回起始下标的元素，调用 Consumer
                @SuppressWarnings("unchecked") E e = (E)list.elementData[i];
                action.accept(e);
                if (list.modCount != expectedModCount)
                    throw new ConcurrentModificationException();
                return true;
            }
            return false;
        }

        public void forEachRemaining(Consumer<? super E> action) {
            int i, hi, mc; // hoist accesses and checks from loop
            ArrayList<E> lst; Object[] a;
            if (action == null)
                throw new NullPointerException();
            if ((lst = list) != null && (a = lst.elementData) != null) {
                if ((hi = fence) < 0) {
                    mc = lst.modCount;
                    hi = lst.size;
                }
                else
                    mc = expectedModCount;
                if ((i = index) >= 0 && (index = hi) <= a.length) {
                    for (; i < hi; ++i) {
                        @SuppressWarnings("unchecked") E e = (E) a[i];
                        action.accept(e);
                    }
                    if (lst.modCount == mc)
                        return;
                }
            }
            throw new ConcurrentModificationException();
        }

        public long estimateSize() {
            return (long) (getFence() - index);
        }

        public int characteristics() {
            return Spliterator.ORDERED | Spliterator.SIZED | Spliterator.SUBSIZED;
        }
    }
```

源码也很短，是一个静态的内部类

我在源码中 trySplit 方法谢了一段注释，这段注释刚好是解释我在上面举得例子，上面为什么这么输出，不懂的可以理解一下这个注释，实际上就是一种分割，控制下标

> 本文首次发布于 [fyypumpkin Blog](http://fyypumpkin.github.io), 作者 [@fyypumpkin](http://github.com/fyypumpkin) ,转载请保留原文链接.