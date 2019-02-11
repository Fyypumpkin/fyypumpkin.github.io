---
layout:     post
title:      Java Semaphore 实现 AQS 分析
subtitle:   了解 AQS 中的共享模式以及信号量
date:       2019-02-11
author:     fyypumpkin
header-img: img/post-bg-debug.png
catalog: true
tags:
    - Java
    - Semaphore
    - AQS
---

## 正文

之前的博客里面已经介绍过 AQS 相关的知识了，之前的 AQS 主要围绕着独占的模式来展开的，今天介绍一种信号量 Semaphore，这个是通过 AQS 中的共享模式实现的

写一个两个队伍排队买票的例子

```java
public class SemaphareTest {
    private volatile AtomicInteger tickets = new AtomicInteger(5);

    class Window implements Runnable {
        private Semaphore semaphore;
        private Integer user;

        public Window(Semaphore semaphore, Integer user) {
            this.semaphore = semaphore;
            this.user = user;
        }

        @Override
        public void run() {
            try {
                semaphore.acquire();
//                里面的并发要自己保证
                if (tickets.getAndDecrement() > 0) {
                    System.out.println("用户【" + user + "】" + "开始买票");
                    Thread.sleep((long) (Math.random() * 1000));
                    System.out.println("用户【" + user + "】" + "买票完成，即将离开");
                    Thread.sleep((long) (Math.random() * 1000));
                    System.out.println("用户【" + user + "】" + "已离开");
                } else {
                    System.out.println("售完");
                }
                semaphore.release();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    private void execute() {
        // 允许两个线程同时进入
        Semaphore semaphore = new Semaphore(2);
        ExecutorService service = Executors.newCachedThreadPool();
        IntStream.range(0, 20).forEach((item) -> {
            service.execute(new Window(semaphore, (item + 1)));
        });
    }

    public static void main(String[] args) {
        new SemaphareTest().execute();
    }
}
```

输出

```java
用户【1】开始买票
用户【2】开始买票
用户【2】买票完成，即将离开
用户【1】买票完成，即将离开
用户【2】已离开
用户【3】开始买票
用户【3】买票完成，即将离开
用户【3】已离开
用户【4】开始买票
用户【4】买票完成，即将离开
用户【1】已离开
用户【5】开始买票
用户【5】买票完成，即将离开
用户【4】已离开
售完
售完
售完
售完
售完
售完
售完
售完
售完
售完
售完
售完
售完
售完
售完
用户【5】已离开
```

上面的程序我模拟了两条队伍服务买票的人员，我通过初始化两个令牌 （Semaphore），拿到令牌的人才能入队买票，否则就需要等待，可以看到，我们票的数量用的是 AtomicInteger，
这里也说明了，在 Semaphore 之间的代码也会有线程安全问题，下面我写一个可能会导致线程安全问题的 demo

```java
public class Semaphore {
    static java.util.concurrent.Semaphore semaphore = new java.util.concurrent.Semaphore(1000);

    private static int count = 0;

    public static void main(String[] args) throws InterruptedException {
        ExecutorService service = Executors.newFixedThreadPool(1000);
        IntStream.range(0, 100000).forEach((item) -> {
            service.execute(() -> {
                try {
                    test2();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });
        });

        Thread.sleep(10000);

        System.out.println(count);

    }


    private static void test2() throws InterruptedException {
        semaphore.acquire();
        count++;
        semaphore.release();
    }
}
```

结果

```java
99995
```

上面程序期望的结果是 100000，然而实际的结果却是五花八门的，因为在 Semaphore 代码块中会有并发问题，这里在写代码的过程中要十分注意。

下面来看一下 Semaphore 是怎么实现的

```java
   public Semaphore(int permits) {
        sync = new NonfairSync(permits);
    }
    
    public Semaphore(int permits, boolean fair) {
        sync = fair ? new FairSync(permits) : new NonfairSync(permits);
    }

```

构造器中传入的 permits 就是令牌数量，构造器会一路调用父类的并将令牌数向上传递，最终传递到 Sync 类中

```java
        Sync(int permits) {
            setState(permits);
        }
```

可以看到，实际上这个令牌数就是当前 AQS 中的 state，之后的操作都是对这个状态进行加减操作

构造器两种入参，一种是令牌数，一种是令牌数加公平模式，默认的情况下是非公平的，这里和 ReentrantLock 一样，最终都是实现的 AQS 同步器，在 Semaphore 中使用的是 AQS 中的共享模式，
本文主要就是通过 Semaphore 讲解一下 AQS 中的共享模式。

先来看一下 Semaphore 中的 acquire 方法

> acquire

```java
    ...
    public void acquire(int permits) throws InterruptedException {
        if (permits < 0) throw new IllegalArgumentException();
        sync.acquireSharedInterruptibly(permits);
    }
    
    public void acquire() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
    ...
```

acquire 方法有好几种入参，我们主要来看一下空入参的，空入参实际上就等于 acquire(1)，内部调用了 acquireSharedInterruptibly 方法，这个方法属于 AQS 中的

```java
    public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }
```

在 AQS 中有会调用子类的 tryAcquireShared(arg) 方法

> tryAcquireShared

```java
        protected int tryAcquireShared(int acquires) {
            return nonfairTryAcquireShared(acquires);
        }
        
        final int nonfairTryAcquireShared(int acquires) {
            for (; ; ) {
//                todo 获取当前的可用状态
                int available = getState();
//                todo 计算剩余可用资源 （可用的减去本次操作需要的资源数，如果小于 0，就会等待 （AQS））
                int remaining = available - acquires;
//                todo 如果可用资源不足或者可用资源足够并且 cas 成功，返回 （如果可用资源不足，会在 AQS 中再次进入 doAcquireSharedInterruptibly 方法）
                if (remaining < 0 ||
                        compareAndSetState(available, remaining))
                    return remaining;
            }
        }
```

这里会执行 cas 操作，如果计算出获取后剩余的令牌小于 0 或者 cas 操作成功，就 return，AQS 中继续向后处理，如果返回的结果是小于 0 的，那么就会进入 AQS 中的 doAcquireSharedInterruptibly
方法

> doAcquireSharedInterruptibly

```java
    private void doAcquireSharedInterruptibly(int arg) {
//        todo 新建一个节点，mode、 = Node.SHARED
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    // todo 尝试获取锁
                    int r = tryAcquireShared(arg);
                    // todo 获取成功
                    if (r >= 0) {
//                        todo 这里获取锁成功后，因为共享，会同时唤醒后续再等待的线程 (链式唤醒后续所有共享的节点)
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
//                    todo 这里和非相应中断的不同
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

这个方法和之前独占模式中的 acquireQueued 方法有一个比较大的区别就是获取锁成功后，共享的会同时唤醒后面一个共享状态的等待线程，而独占模式只是设置当前的 head，对比一下独占模式的 acquireQueued 方法

> acquireQueued

```java
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                // todo 检查前置node，如果是 head， 表示自己可以尝试获取锁， head标识当前拥有锁的线程 （公平的体现）
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    // todo 获取成功，就把自己设置 head
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                // todo 获取锁失败，则通过 shouldParkAfterFailedAcquire 方法来标记当前线程是否需要阻塞，需要的话，调用 parkAndCheckInterrupt 方法来阻塞中断
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    // todo 等待下个线程唤醒后继续执行
                    // todo 这里中断不是立即执行的，还是会等待获取到锁
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

来看一下共享模式下的 setHeadAndPropagate 的实现

> setHeadAndPropagate

```java

```

> doReleaseShared

```java
    private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; // Record old head for check below
        setHead(node);
       
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
//            todo 如果后继节点是 shared 的，就会进入 doReleaseShared 方法
            Node s = node.next;
            if (s == null || s.isShared())
                doReleaseShared();
        }
    }
```

```java
    private void doReleaseShared() {
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
//                    todo 主要关注这一行代码，如果节点状态是 SIGNAL （说明后续节点需要被唤醒）
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    unparkSuccessor(h);
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }
```

可以看到，在成功获取到共享锁后，会同时唤醒后续的一个节点中等待的线程（如果后续是共享模式的并且头结点状态是 SIGNAL （说明有线程在等待） 的）。

下面来看一下 Semaphore 的 release 方法

> release

```java
    public void release() {
        sync.releaseShared(1);
    }
```

release 同样是 AQS 中的方法

> releaseShared

```java
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
//            todo 释放锁的时候，唤醒后续等待节点
            doReleaseShared();
            return true;
        }
        return false;
    }
```

> tryReleaseShared

```java
        protected final boolean tryReleaseShared(int releases) {
//            todo 确保成功
            for (; ; ) {
                int current = getState();
//                todo 对当前的状态加上 releases
                int next = current + releases;
                if (next < current) // overflow
                    throw new Error("Maximum permit count exceeded");
                if (compareAndSetState(current, next))
                    return true;
            }
        }
```

如果调用 tryReleaseShared 成功，就会调用 doReleaseShared 方法唤醒后续正在等待的节点

这里的唤醒我觉得是有一种链式的效果的，第一个节点释放锁，唤醒后续的节点，后续的节点获取到锁后又会唤醒后续的节点（共享模式的节点），所以一个读写锁，写锁后面排队排着一万个读锁，写锁释放后，后续的一万个读锁都能获取到锁并执行,下面是一个例子

```java
    public static void main(String[] args) {
        new Thread(() -> {
            lock.writeLock().lock();
            try {
                System.out.println(System.currentTimeMillis());
                Thread.sleep(2000);
                System.out.println(System.currentTimeMillis());
                System.out.println();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            lock.writeLock().unlock();
        }).start();

        new Thread(() -> {
            lock.readLock().lock();
            System.out.println("read 1");
            System.out.println(System.currentTimeMillis());
            lock.readLock().unlock();
        }).start();

        new Thread(() -> {
            lock.readLock().lock();
            try {
                Thread.sleep(1000);
                System.out.println(System.currentTimeMillis());
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("read 2");
            lock.readLock().unlock();
        }).start();


        new Thread(() -> {
            lock.readLock().lock();
            try {
                Thread.sleep(2000);
                System.out.println(System.currentTimeMillis());
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("read 3");
            lock.readLock().unlock();
        }).start();
    }
```

执行结果

```java
1549884578856
1549884580860

read 1
1549884580861
1549884581864
read 2
1549884582864
read 3
```

可以看到，三个读锁之间的打印都相隔 1 秒左右，可以说明，三个读锁是同时拿到的，下一篇博客就会讲到读写锁的实现

> 本文首次发布于 [fyypumpkin Blog](http://fyypumpkin.github.io), 作者 [@fyypumpkin](http://github.com/fyypumpkin) ,转载请保留原文链接.