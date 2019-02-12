---
layout:     post
title:      Java ReentrantReadWriteLock 实现 AQS 分析
subtitle:   了解 AQS 中的共享模式以及读写锁
date:       2019-02-12
author:     fyypumpkin
header-img: img/post-bg-debug.png
catalog: true
tags:
    - Java
    - ReentrantReadWriteLock
    - AQS
---

## 正文

今天主要来看一下 Java 中的读写锁 `ReentrantReadWriteLock`，读写锁也是可重入锁的一种，可以看出，其实现也是依托于 AQS 来实现的，可以说 AQS 就是并发包的核心知识，了解了 AQS，并发包中一半的内容也就很容易了解了

读写锁在平时我是用的不多，可能受限于经验以及经历，读写锁的了解我是直接来源于其他博客以及对于源码的阅读，我主要会以源码的角度分析一下读写锁的原理，上一篇博客的末尾我也使用了读写锁的例子，说明了一下共享模式下唤醒的方式 （链式的），同属于一个共享模式下等待的线程会和多米诺骨牌一样，一个被前一个唤醒。

读写锁顾名思义，分读锁和写锁，读锁是共享的，写锁是独占的，也就是说，读写锁用到了 AQS 里面独占模式以及共享模式，读写锁我们分开解析，先来看看构造器。

```java
    public ReentrantReadWriteLock() {
        this(false);
    }
    public ReentrantReadWriteLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
//        todo 入参用于传递 sync
        readerLock = new ReadLock(this);
        writerLock = new WriteLock(this);
    }
```

构造器比较简单，空入参或者包含是否公平的入参，默认情况下是非公平的，我们主要讲解一下非公平的，公平的有兴趣可以自己看一看源码，变化不大。

读写锁共用了一个 32 位 state，将高 16 位作为了读锁的状态，低 16 位作为写锁的状态，通过两个方法分别获取读写锁的次数

```java
    static int sharedCount(int c) {
    // todo 高 16 位 （读锁的次数）
        return c >>> SHARED_SHIFT;
    }

    static int exclusiveCount(int c) {
    // todo 低 16 位 （写锁的次数）
        return c & EXCLUSIVE_MASK;
    }
```

先来看一下写锁，写锁就是一个独占锁

> lock

```java
    public void lock() {
        // AQS
        sync.acquire(1);
    }
```

看过这么多遍 AQS 了，我们直接去看 tryAcquire 方法

> tryAcquire

```java
protected final boolean tryAcquire(int acquires) {
    Thread current = Thread.currentThread();
    int c = getState();
    // todo 写锁的重入次数
    int w = exclusiveCount(c);
    // todo 有读写锁
    if (c != 0) {
        // (Note: if c != 0 and w == 0 then shared count != 0)
        // todo 如果没有写锁 （因为锁不等于0，没有写锁表明有读锁，返回 false， 进 AQS队列） 或者当前线程不等于写锁的线程，就返回 false
        // todo 返回 false 后进入 AQS 队列等待
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // Reentrant acquire
        // todo 上面已经确保了是当前线程，所以不需要 cas
        setState(c + acquires);
        return true;
    }

    // todo writerShoudBlock 返回false，这里主要用于判断 cas 是否成功，有没有其他线程抢占，失败进 AQS 队列
    if (writerShouldBlock() ||
            !compareAndSetState(c, c + acquires))
        return false;
    setExclusiveOwnerThread(current);
    return true;
}
```

上面代码中，出现了一个 writerSouldBlock 方法，这个方法主要和公平非公平有关

```java
    /**
    * todo 和 writerShouldBlock 类似，只不过非公平模式下，为了防止写线程饥饿
    */
    abstract boolean readerShouldBlock();

    /**
    * todo 这里主要会和公平以及非公平策略挂钩，非公平直接返回 false，公平模式判断当前是否允许获取锁
    */
    abstract boolean writerShouldBlock();

    ... 公平模式实现
    final boolean writerShouldBlock() {
        return hasQueuedPredecessors();
    }

    final boolean readerShouldBlock() {
        return hasQueuedPredecessors();
    }

    public final boolean hasQueuedPredecessors() {
    // The correctness of this depends on head being initialized
    // before tail and on head.next being accurate if the current
    // thread is first in queue.
    // todo 用于判断当前线程是否是下一个可以获取锁的线程
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
    }

    ... 非公平模式实现
    final boolean writerShouldBlock() {
        return false; // writers can always barge
    }

    final boolean readerShouldBlock() {
        return apparentlyFirstQueuedIsExclusive();
    }

    final boolean apparentlyFirstQueuedIsExclusive() {
    Node h, s;
    // todo (head != null) && (head.next != null) && (head.next 不是共享锁 （是写锁的时候，并且线程不为空）)
    // todo 这里保证了当一个持锁线程的后续线程是写线程的时候，那么读线程就要等待，否则可以尝试重入，防止了
    // todo 当前线程 -> 写线程 -> (读线程获取锁) 最后的读线程需要阻塞，防止写线程饥渴 （读 >> 写）
    return (h = head) != null &&
        (s = h.next)  != null &&
        !s.isShared()         &&
        s.thread != null;
}
```

写锁释放方法 unlock

> unlock

```java
    public void unlock() {
        sync.release(1);
    }
```

> tryRelease

```java
    protected final boolean tryRelease(int releases) {
        // todo 必须是独占锁
        if (!isHeldExclusively())
            throw new IllegalMonitorStateException();
        int nextc = getState() - releases;
        // todo 因为可重入，所以全部释放完才算锁释放
        boolean free = exclusiveCount(nextc) == 0;
        if (free)
            setExclusiveOwnerThread(null);
        setState(nextc);
        return free;
    }
```

下面看一下读锁的上锁以及释放的过程

> lock

```java
    public void lock() {
        sync.acquireShared(1);
    }
```

> acquireShared AQS

```java
    public final void acquireShared(int arg) {
        // todo 如果获取的锁小于 0 （获取失败）
        if (tryAcquireShared(arg) < 0)
        // todo 进入 AQS 自旋未果后挂起等待唤醒 (共享模式会在获取到锁也唤醒后续节点)
            doAcquireShared(arg);
    }
```

> tryAcquireShared

```java
protected final int tryAcquireShared(int unused) {
    Thread current = Thread.currentThread();
    int c = getState();
    // todo 有写锁并且当前线程不等于写线程，返回 -1 进入 AQS 队列 (当前线程的读锁是可以上锁的 （锁降级）)
    if (exclusiveCount(c) != 0 &&
            getExclusiveOwnerThread() != current)
        return -1;
    // todo 读锁的重入次数
    int r = sharedCount(c);
    // todo 判断当前持有锁线程的后面一个节点是否写线程，是写线程的话就返回 true （非公平）
    if (!readerShouldBlock() &&
            r < MAX_COUNT &&
            compareAndSetState(c, c + SHARED_UNIT)) {
        if (r == 0) {
            firstReader = current;
            firstReaderHoldCount = 1;
        } else if (firstReader == current) {
            // todo 重入
            firstReaderHoldCount++;
        } else {
        // todo 每个线程的读锁次数，通过 ThreadLocal 实现，这里的 cachedHoldCounter 是上一次读操作的缓存，
        // todo firstReader 和 firstReaderHoldCount 以及 cachedHoldCounter都是提升效率，减少 ThreadLocal 的访问
            HoldCounter rh = cachedHoldCounter;
            if (rh == null || rh.tid != getThreadId(current))
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0)
                readHolds.set(rh);
            rh.count++;
        }
        return 1;
    }

    // todo 上面加锁失败就进入死循环，这里面的逻辑和外面实际上差不多，只是不断循环直到又返回结果，就不贴出来了
    return fullTryAcquireShared(current);
}

```

> unlock

```java
    public void unlock() {
        sync.releaseShared(1);
    }
```

> releaseShared

```java
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
        // todo 释放锁的时候，唤醒后续等待节点
            doReleaseShared();
            return true;
        }
        return false;
    }
```

> tryReleaseShared

```java
protected final boolean tryReleaseShared(int unused) {
    Thread current = Thread.currentThread();
    // todo 获取相应的重入数，并且计算
    if (firstReader == current) {
        // assert firstReaderHoldCount > 0;
        if (firstReaderHoldCount == 1)
            firstReader = null;
        else
            firstReaderHoldCount--;
    } else {
    // todo 非本线程就拿上一次 cache 的 counter，如果不是再从 ThreadLocal 里面拿，然后做相应的计算
        HoldCounter rh = cachedHoldCounter;
        if (rh == null || rh.tid != getThreadId(current))
            rh = readHolds.get();
        int count = rh.count;
        if (count <= 1) {
            readHolds.remove();
            if (count <= 0)
                throw unmatchedUnlockException();
        }
        --rh.count;
    }
    // todo 对 state 进行值得修改
    for (; ; ) {
        int c = getState();
        int nextc = c - SHARED_UNIT;
        // todo 确保成功
        if (compareAndSetState(c, nextc))
            // Releasing the read lock has no effect on readers,
            // but it may allow waiting writers to proceed if
            // both read and write locks are now free.
            return nextc == 0;
    }
}
```

关于锁降级，读写锁有一种锁降级的写法，就是获取写锁，获取读锁，释放写锁，在释放读锁

> 锁降级

```java
try{
    write.lock();
    doSomeThing();
    read.lock();
    write.unlock();
    doSomeReadThing();
}finally{
    read.unlock();
}
```

说一下这种写法的个人理解吧，网上有很多说法，这里我就不说了，单纯的说一下自己的理解，假设我们不这么写，而是释放读锁之后在获取读锁，这样会有什么问题呢？就是在获取读锁的时候，可能被其他线程抢先获取了写锁，进而本线程无法获取读锁，假设释放完写锁我们需要对上面修改的数据进行一些操作，就会被分割开，下次再拿到锁的时候，数据就可能被其他线程修改了就不是预期的了，有人可能会说，整个操作都放在写锁里面，这样固然没有问题，但是锁住的代码就会比较多，写锁是一种独占锁，就会导致并发搞得情况效率比较低，所以这里就事先获取了读锁，即使在写锁释放了的情况，其他写线程也无法获取锁，而其他读的线程就可以获取读锁了，提高了效率。

上面也只是我个人对于锁降级的看法，可能有些地方不太得当或者不太正确，大家也应有自己的理解。

> 本文首次发布于 [fyypumpkin Blog](http://fyypumpkin.github.io), 作者 [@fyypumpkin](http://github.com/fyypumpkin) ,转载请保留原文链接.