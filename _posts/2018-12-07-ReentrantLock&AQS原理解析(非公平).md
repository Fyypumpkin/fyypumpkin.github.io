---
layout:     post
title:      Java ReentrantLock AQS 实现(非公平)（一）
subtitle:   了解 ReentrantLock 非公平实现锁的获取和释放
date:       2018-12-07
author:     fyypumpkin
header-img: img/post-bg-2015.jpg
catalog: true
tags:
    - Java
    - AQS
    - J.U.C
    - ReentrantLock
---

## 正文

很久没有写博客了，最近看（炒冷饭）了一下 Java 中 ReentrantLock 锁的实现，ReentrantLock 锁的逻辑大部分都在 AQS （AbstractQueuedSynchronizer）里面，所以本文也大致在介绍 AQS 中的实现

### 先来看看锁中初级的用法

> 来一段简单的代码

```java
class XXX {
        ...
        ReentrantLock lock = new ReentrantLock();
    
        private void test() {
            lock.lock();
            doSomeThing();
            lock.unlock();
        }
        ...
    }
```

这段代码简单的不能再简单了，在一个类中我们实例化了一个 ReentrantLock 锁，在方法中调用，在执行相关并发代码前后进行加锁和解锁的操作，在同一个 XXX 对象中的时候，这个代码就是线程安全的，那么 ReentrantLock 是如何保证中间的 doSomething 是线程安全的呢？
我们就直接看 ReentrantLock 的源码，由于存在公平和非公平的实现，我们常用的是非公平的实现，我这里主要就会分析一下非公平的实现。

### 初探 ReentrantLock 源码


> 先来看看 ReentrantLock 的构造器

```java
    public ReentrantLock() {
        sync = new NonfairSync();
    }
```

可以看到， ReentrantLock 默认使用的是非公平 Sync 的实现，看一下 Sync 的继承图

![](/assets/img/NonFair.png)

NonFairSync -> Sync -> AbstractQueuedSynchronizer -> AbstractOwnableSynchronizer

AbstractOwnableSynchronizer 内部几乎没有逻辑，本文就不深入了

了解了整个继承链路后，我们就可以开始从我们常用的方法作为切入点，开始阅读源码

#### 加锁

> ReentrantLock 中的 lock() 方法

```java
    public void lock() {
        sync.lock();
    }
```

ReentrantLock 中的 lock() 直接将整个逻辑托付给了 Sync 类，这里的 Sync 就是刚才的 NonFairSync

> NonFairSync 类

```java
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        /**
         * Performs lock.  Try immediate barge, backing up to normal
         * acquire on failure.
         */
        final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }

        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    }
```

在 NonFaitSync 中，非公平的实现会直接尝试通过 CAS 操作获取一次锁，这里的 CAS 操作是在 AQS 中实现的，在 AQS 中会保存一个状态 `state` 

```java
    /**
     * The synchronization state.
     */
    private volatile int state;
```

可以看出来， AQS 通过这个 `state` 来标识当前的同步状态，初始为 0 ，表示没有人占用当前锁，则当前线程通过 `compareAndSetState(0, 1)` 来尝试获取锁， 获取成功后，就直接将当前线程设置为独占
否则，就会调用 `acquire` 方法，`acquire` 方法的实现也是在 AQS 中。

```java
   public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

在 `acquire` 方法中，回去调用 `tryAcquire`， `addWaiter`，和 `acquireQueued` 方法，同时 AQS 又通过模板的形式，将 `tryAcquire` 方法通过子类来实现， 我们直接来看子类 `Sync` 中实现的 `nonfairTryAcquire`
（这里有点不理解，为什么不直接在 `NonFairSync` 中直接实现 `tryAcquire` 方法，而是调用了父类的 `nonfairTryAcquire`）

```java
 final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```

这个方法也很简单，首先会获取当前线程，同时获取当前锁的占用状态， 如果当前锁状态 == 0，说明当前锁没有被占用， 则去尝试通过 CAS 获取一次锁，否则说明有线程已经占用了当前的锁（即存在竞争），
那么久获取当前所拥有锁的线程和当前的线程是否是同一个，如果是同一个，则表示锁重入了，就将状态加一（这里是实现锁重入的关键），很容易想到，在释放锁的时候，也会将状态减一，当减到 0 时，才表示锁
释放，所以我们使用锁应该都要成对出现，否则就容易出现死锁。

如果当前状态既不是 0 ，也不是锁的重入，那么就返回获取锁失败，此时 `acquire` 方法中的判断条件就会进入下一个 `acquireQueued(addWaiter(Node.EXCLUSIVE), arg))`，先来看一下 `addWaiter`

```java
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        // 不需要初始化，执行一次 CAS 加队列
        if (pred != null) {
            node.prev = pred;
            // 将当前 node 加入队尾（可能由于并发原因失败）
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        // 上面执行失败，进入 enq 方法，循环执行
        enq(node);
        return node;
    }
```

`Node.EXCLUSIVE` 实际上就是 `null`， 用于将 Node 中的 `nextWaiter` 设置为 `null`，表示独占
初次进入 `addWaiter` 方法中，AQS 维护的队列实际上是空的，就会直接进入 `enq` 方法

```java
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                // 将node添加到队尾
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

`enq` 方法会包含队列的初始化，将队列的 `head` 设置为一个 `new Node`, 然后再将上面传进来的 `node` 接入到队尾， 然后返回，由于这两个操作都可能会因为并发问题而失败，所以放在了一个死循环中执行
执行成功后就会 return，最后 `addWaiter` 方法就会返回一个 node，返回后就会进入 `acquireQueued` 方法

```java
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                // 检查前置node，如果是 head， 表示自己可以尝试获取锁， head标识当前拥有锁的线程 （公平的体现）
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    // 获取成功，就把自己设置 head
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                // 获取锁失败，则通过 shouldParkAfterFailedAcquire 方法来标记当前线程是否需要阻塞，需要的话，调用 parkAndCheckInterrupt 方法来阻塞中断
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    // 等待下个线程唤醒后继续执行
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

进入 `acquireQueued` 方法，同样是一个死循环，循环中会去获取 `node` 的前置节点，如果前置节点是 head 节点，就表示自己试最先可以开始尝试获取锁的线程 （公平的体现，当然，前面非公平优先级会更高）
那么就会调用上面的 `tryAcquire` 方法进行获取锁，同样的 `tryAcquire` 获取锁也不一定会成功，因为非公平下，新线程首次可以直接尝试获取锁，而不需要进行排队。
如果本次获取成功，那么就会将当前 node 设置为 head 节点， 同时将上一个 head 节点设置为 `null`， 可以看到注释中说了，可以帮助 GC，发生 GC 时，上一个 head 节点就会被回收。
如果本次获取锁失败了，那么就会进入 `shouldParkAfterFailedAcquire` 方法

```java
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus; //0
        // 如果前节点是 SINGAL 状态，则表示后面的需要阻塞
        if (ws == Node.SIGNAL)
            return true;
//            ws > 0 表示状态时 CANCELED，通过循环将当前节点之前所有取消状态的节点移出队列；
        if (ws > 0) {
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;

            // 前节点是其他状态，当前节点设置为 SIGNAL
        } else {
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```

进入 `shouldParkAfterFailedAcquire` 方法后，首先会获取前驱节点的 `waitStatus`， 初始化 `Node` 默认是 0，首次进入该方法，会直接跳转到 `compareAndSetWaitStatus(pred, ws, Node.SIGNAL);`， 这个语句
会将前驱节点的 `waitStatus` 状态设置为 `SIGNAL`， 然后返回 `false`，返回后，又会进入 `acquireQueued` 中的死循环，同样的会在执行上面流程一次，若没获取到锁，那么再次进入 `shouldParkAfterFailedAcquire` 方法
这次进入，前驱节点的 `waitStatus` 就会是 `SIGNAL`，就会返回 `true`，返回后就会进入 `parkAndCheckInterrupt` 方法

```java
    private final boolean parkAndCheckInterrupt() {
        // 等待
        LockSupport.park(this);
        return Thread.interrupted();
    }
```

进入该方法后，就会滴啊用 `LockSupport` 的 `park` 方法将当前线程阻塞调，等待其他线程来唤醒，这样整个加锁操作就完成了

#### 解锁

下面来看一下 `unlock` 方法
同样的， `unlock` 方法也会交给 Sync 来完成，我们直接来看 Sync 中的 `tryRelease` 方法

```java
      protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
```

可以看到，解锁会对所有重入的锁都进行解锁完成才认为是解锁完成

```java
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```

在 AQS 中，会调用 `tryRelease` 方法，当所有重入的锁都被释放后，获取当前所在的 head，判断 head 不是 `null` 并且 head 的 `waitStatus` 不为 0 ，就去唤醒后续节点，不为 0 的原因是，如果当前节点为0，说明没有竞争，也就没有等待的节点
无需去唤醒，否则表示有过竞争，则需要相应的唤醒操作

下面来看一下 `unparkSuccessor`

```java
    private void unparkSuccessor(Node node) {
        /*
         *  1 CANCELLED
         * -1 SIGNAL
         * -2 CONDITION
         * -3 PROPAGATE
         */
        int ws = node.waitStatus;
        // 更新节点状态
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);
//        拿节点的下一个节点， 如果节点是 null，那就从后往前拿，拿到最靠近前节点的一个状态不是 CANCELLED 的，然后执行唤醒
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```

这个方法中，同样会去判断一次状态，并将后续一个状态不为 CANCELLED 的节点进行唤醒，唤醒后，相应的线程就会在之前 park 住的地方继续向下执行

其实在 AQS 中，就是维护这样一个等待的队列，来控制加锁解锁



> 本文首次发布于 [fyypumpkin Blog](http://fyypumpkin.github.io), 作者 [@fyypumpkin](http://github.com/fyypumpkin) ,转载请保留原文链接.