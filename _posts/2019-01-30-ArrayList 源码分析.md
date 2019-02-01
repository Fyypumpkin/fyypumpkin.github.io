---
layout:     post
title:      Java ArrayList 源码分析
subtitle:   了解 ArrayList 源码及其工作原理
date:       2019-01-30
author:     fyypumpkin
header-img: img/post-bg-digital-native.jpg
catalog: true
tags:
    - Java
    - ArrayList
    - Java 集合
---

## 正文

ArrayList 在我们的代码中相信一定是很常见的，今天就来看一看 ArrayList 的源码 （不包含 Spliterator 相关的， Spliterator 会在后面单独讲一下），这次博客主要内容直接写在代码注释里面

来看一下 ArrayList 的相关变量

```java
 private static final long serialVersionUID = 8683452581122892189L;
    // todo 默认的大小
    private static final int DEFAULT_CAPACITY = 10;

    // todo 这个空数组用于构造器中传入初始容量 = 0
    private static final Object[] EMPTY_ELEMENTDATA = {};

//   todo  这个数组用于默认构造器，（会使用默认的容量 10）
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

//   todo 真实的数组元素
    transient Object[] elementData; // non-private to simplify nested class access

//   todo 数组大小
    private int size;
    
    // todo 最大数组容量 （这里实际上最大的还是 Integer.MAX_VALUE, 为了保证某些虚拟机会在数组中存放一些头信息，所以 -8）
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
```

看到变量里面有两个空的数组，用处是不一样的，一个是构造器传入容量为 0 的时候使用，另一个是默认构造器初始化使用，默认构造器是有默认容量的，为 10，当第一次使用，发现是默认构造器初始化的就会初始化数组长度为 10

看一下下面三个构造器

```java
// 带初始容量的构造器，可以看到，当传入的容量为 0 的时候，会使用 EMPTY_ELEMENTDATA 进行赋值，否则就至今进行数组的初始化
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }
// 默认的构造器，无参数，直接将数组元素设置为 DEFAULTCAPACITY_EMPTY_ELEMENTDATA  
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }

// 将另一个集合复制进来的构造器，使用了 System.copyOf 函数，如果集合是空的，那么就使用 EMPTY_ELEMENTDATA   
    public ArrayList(Collection<? extends E> c) {
    // 转换成数组
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // replace with empty array.
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }    
```
add 和 set 方法

```java
    public boolean add(E e) {
//        todo 确保容量，不足就扩容并且会增加 modCount，在迭代过程中，不允许 add 操作
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }
    
    public void add(int index, E element) {
//        todo 确保 index 合法， index <= size && index >= 0
        rangeCheckForAdd(index);

        ensureCapacityInternal(size + 1);  // Increments modCount!!
//        todo 调用 arrayCopy 方法法复制 index 后面的元素向后移动一位
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        elementData[index] = element;
        size++;
    }
    
    // set 方法并不会改变 modCount， 所以在使用迭代器迭代的时候，可以通过 set 修改数据（涉及到数组长度的操作都会修改 modCount）
    public E set(int index, E element) {
        rangeCheck(index);
//        todo 对数组制定下表的元素重新赋值，返回老元素，不会改变迭代器 modCount 状态
        E oldValue = elementData(index);
        elementData[index] = element;
        return oldValue;
    }

```

在上面的 add 方法中，有一个 ensureCapacityInternal 方法，该方法主要用于扩容的判断

```java

    private void ensureCapacityInternal(int minCapacity) {
        // todo 这里就会使用到，当构造器不传和传 0 处理方式是不一样的，如果是空构造，就会使用 DEFAULT_CAPACITY 和 minCapacity 中大的作为初始容量
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        ensureExplicitCapacity(minCapacity);
    }

    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // overflow-conscious code
        // todo 如果 minCap 大于当前的长度，就需要扩容
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }

    private void grow(int minCapacity) {
        // overflow-conscious code
//        todo 记录老的长度
        int oldCapacity = elementData.length;
//        todo newCapacity = 1.5 * oldCapacity
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        // 我认为这里是越界了
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
//        todo 如果大于 MAX_ARRAY_SIZE （Integer.MAX - 8）,就会使用 Integer.MAX
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        // todo 调用 copy 函数
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
    
```

再来看一下 get 方法，get 方法比较简单，范围判断后直接返回对应数组下标

```java
    E elementData(int index) {
//        todo 直接返回数组的第 index 个元素，不会改变 modCount 状态
        return (E) elementData[index];
    }

    public E get(int index) {
    // 范围检查
        rangeCheck(index);
        return elementData(index);
    }
```

toArray 方法

```java
     public Object[] toArray() {
//        todo 返回一个新的数组
        return Arrays.copyOf(elementData, size);
     }

    @SuppressWarnings("unchecked")
    public <T> T[] toArray(T[] a) {
//        todo 如果传进来的数组长度太小，就会使用新建一个数组
        if (a.length < size)
            // Make a new array of a's runtime type, but my contents:
            return (T[]) Arrays.copyOf(elementData, size, a.getClass());
//        todo 长度够，就在传进来的数组上赋值
        System.arraycopy(elementData, 0, a, 0, size);
        if (a.length > size)
            a[size] = null;
        return a;
    }
```

和删除相关的代码 remove fastRemove clear

```java
  public E remove(int index) {
        rangeCheck(index);

        modCount++;
        E oldValue = elementData(index);
//        todo 计算需要移动的元素个数
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
    }

    public boolean remove(Object o) {
//        todo null 要特殊处理
        if (o == null) {
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index);
                    return true;
                }
        } else {
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }

 
//    todo 和 remove(index) 功能一样
    private void fastRemove(int index) {
        modCount++;
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work
    }

    public void clear() {
//      todo 清空所有元素，并且 size = 0；
        modCount++;

        // clear to let GC do its work
        for (int i = 0; i < size; i++)
            elementData[i] = null;

        size = 0;
    }
    
    // 范围移除
    protected void removeRange(int fromIndex, int toIndex) {
        modCount++;
        // 需要移动的元素个数
        int numMoved = size - toIndex;
        System.arraycopy(elementData, toIndex, elementData, fromIndex,
                         numMoved);

        // clear to let GC do its work
        int newSize = size - (toIndex-fromIndex);
        for (int i = newSize; i < size; i++) {
            elementData[i] = null;
        }
        size = newSize;
    }    

```

看一下 ArrayList 的迭代器

```java
    private class Itr implements Iterator<E> {
        int cursor;       // index of next element to return
        int lastRet = -1; // index of last element returned; -1 if no such
        int expectedModCount = modCount;

        // 通过游标是否到达 size 来判断有没有下一个元素
        public boolean hasNext() {
            return cursor != size;
        }

        @SuppressWarnings("unchecked")
        public E next() {
        // 判断 modCount 是否被修改，被修改就会抛出异常
            checkForComodification();
            int i = cursor;
            if (i >= size)
                throw new NoSuchElementException();
            Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length)
                throw new ConcurrentModificationException();
                // 游标+1
            cursor = i + 1;
            return (E) elementData[lastRet = i];
        }

        // todo 在迭代器中要使用迭代器中的 remove 方法
        public void remove() {
            if (lastRet < 0)
                throw new IllegalStateException();
            checkForComodification();

            try {
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
            // todo 这里会检查 遍历中的 modCount 和期望的是否一致，不一致会抛出异常
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
    }

```