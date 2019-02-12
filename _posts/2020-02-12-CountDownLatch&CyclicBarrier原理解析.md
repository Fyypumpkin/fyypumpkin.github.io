---
layout:     post
title:      Java CountDownLatch && CyclicBarrier原理解析
subtitle:   了解 CountDownLatch 以及 CyclicBarrier
date:       2019-02-12
author:     fyypumpkin
header-img: img/post-bg-debug.png
catalog: true
tags:
    - Java
    - CountDownLatch
    - CyclicBarrier
    - AQS
---

## 正文

CountDownLatch 和 CyclicBarrier 也是第一次接触，当然，这两个类也比较简单，源码量非常少（除去 AQS），先来看看这两个类怎么用吧

> CountDownLatch 简单用法 - 控制线程一起执行

```java
    public static void main(String[] args) throws InterruptedException {
        CountDownLatch beginCount = new CountDownLatch(1);
        for (int i = 0; i < 10; i++) {
            final int num = i;
            new Thread(() -> {
                try {
                    beginCount.await();
                    System.out.println(num + " " + System.currentTimeMillis() / 1000);
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }).start();
        }

        System.out.println(System.currentTimeMillis() / 1000 + "\n");
        Thread.sleep(1000);
        beginCount.countDown();
    }
```
结果

```java
1549969191

0 1549969192
2 1549969192
3 1549969192
1 1549969192
8 1549969192
7 1549969192
6 1549969192
5 1549969192
4 1549969192
9 1549969192
```

上面结果可以看到，10个线程打印出数字的时间均为 1549969192，相比我调用 `countDown` 前打印相差一秒。
10个线程执行的时候都遇到了 `await` 方法，因此都等待 `countDown` 的触发，当执行到 `countDown` 的时候，是个线程就一起执行了

> CountDownLatch 简单用法 - 所有线程结束后结束退出

```java
public static void main(String[] args) throws InterruptedException {
    CountDownLatch countDownLatch = new CountDownLatch(2);
    new Thread(() -> {
            try {
                Thread.sleep(1000);
                System.out.println(System.currentTimeMillis() / 1000 + " - 1");
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                countDownLatch.countDown();
            }
        }).start();

        new Thread(() -> {
            try {
                Thread.sleep(2000);
                System.out.println(System.currentTimeMillis() / 1000 + " - 2");
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                countDownLatch.countDown();
            }
        }).start();

        countDownLatch.await();
        System.out.println(System.currentTimeMillis() / 1000);
    }
```

结果

```java
1549969496 - 1
1549969497 - 2
1549969497
```

上述程序，main 方法被 `await` 阻塞，知道所有的线程内都执行完毕了才退出 main 方法，每个线程执行完后，都调用一次 `countDown` 方法，知道初始化的数量使用完毕，`await` 方法就会放行，看似很神奇，实际上原理也很简单
 
看一下 CountDownLatch 中的方法

![](/assets/img/CountDownLatch.png)

源码非常简单，来看看构造器，构造器也只有一种

> 构造器

```java
   public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }
```

> await

```java
    public void await() throws InterruptedException {
        // todo AQS 方法
        sync.acquireSharedInterruptibly(1);
    }
```

> acquireSharedInterruptibly

```java
    public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }
```

> tryAcquireShared

```java
    protected int tryAcquireShared(int acquires) {
        // todo 这个方法只会判断当前的 state 是不是 == 0，不是就返回 -1，进入 AQS 队列阻塞，是的话就直接返回 0 ，调用结束
        // todo 初始化 == 0 的时候，await 方法不起作用的
        return (getState() == 0) ? 1 : -1;
    }
```

> countDown

```java
    public void countDown() {
        // todo AQS 方法
        sync.releaseShared(1);
    }
```

> releaseShared

```java
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```

> tryReleaseShared

```java
    protected boolean tryReleaseShared(int releases) {
        // Decrement count; signal when transition to zero
        // todo 如果返回 true，就会执行 AQS 的 doReleaseShared 方法进行唤醒后续共享等待的节点
        for (;;) {
            int c = getState();
            // todo 先获取当前状态 (初始化 == 0 的时候，countDown 方法直接返回的)
            if (c == 0)
                return false;
            int nextc = c-1;
            // todo cas 操作，如果递减后 == 0，返回 true。对后续等待节点进行唤醒
            if (compareAndSetState(c, nextc))
                return nextc == 0;
        }
    }
```

上面很多方法都是 AQS 中的共享模式下的一些方法，不懂的可以看一下前几期的博客 （ReentrantReadWriteLock 和 Semaphore），上面的源码很容易理解，调用  `await` 方法的时候，实际上是对 state 的判断，是否 == 0，如果不是的话就进入 AQS 等待， 调用 `countDown` 方法的时候，实际上就是对 state 进行递减操作，当减到 0 的时候，`return nextc == 0` 就会返回 `true`，那么就会触发 AQS 中的 `doReleaseShared` 方法唤醒后续等待的节点，因为是共享的，链式唤醒，所有等待的线程都会被唤醒开始执行，此时 `tryAcquireShared` 就会返回 1，直接往后执行（相当于获取到锁了）。

CountDownLatch 原理很简单，还有一个类功能类似的，就是 CyclicBarrier，字面意思就是一种屏障，看一下类似的用法

> CyclicBarrier 简单用法 - 线程同时运行

```java
private static void test4() throws InterruptedException {
    CyclicBarrier cyclicBarrier = new CyclicBarrier(2);

    new Thread(() -> {
        try {
            cyclicBarrier.await();
            System.out.println(System.currentTimeMillis() / 1000);
        } catch (Exception e) {
            e.printStackTrace();
        }

    }).start();

    Thread.sleep(1000);

    new Thread(() -> {
        try {
            cyclicBarrier.await();
            System.out.println(System.currentTimeMillis() / 1000);
        } catch (Exception e) {
            e.printStackTrace();
        }

    }).start();
}
```

输出

```java
1549971516
1549971516
```

同时，CyclicBarrier 初始化的时候，还可以带一个 Runnable 参数，用于屏障使用完毕，在所有线程执行前执行的一个任务

> CyclicBarrier Runnable 测试

```java
    private static void test5() {
        CyclicBarrier cyclicBarrier = new CyclicBarrier(1, () -> {
            System.out.println("我是前置任务");
        });

        new Thread(() -> {
            try {
                cyclicBarrier.await();
                System.out.println("我是屏障后待执行的线程");
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (BrokenBarrierException e) {
                e.printStackTrace();
            }
        }).start();
    }
```

结果

```java
我是前置任务
我是屏障后待执行的线程
```

可以看到，我自己的线程在执行前，先执行了 CyclicBarrier 传入的任务

从上面的测试例子来看，CyclicBarrier 至少有两种构造器，事实上就是两种

> 构造器

```java
    /** The lock for guarding barrier entry */
    // todo 锁
    private final ReentrantLock lock = new ReentrantLock();
    /** Condition to wait on until tripped */
    // todo 条件队列
    private final Condition trip = lock.newCondition();
    /** The number of parties */
    // todo 屏障数量 （线程数量）
    private final int parties;
    /* The command to run when tripped */
    // todo 前置任务
    private final Runnable barrierCommand;
    /** The current generation */
    // todo 代，当线程数到达屏障数量，就会执行所有线程，并且更新换代（进入下一轮使用）
    private Generation generation = new Generation();

    public CyclicBarrier(int parties, Runnable barrierAction) {
        if (parties <= 0) throw new IllegalArgumentException();
        this.parties = parties;
        this.count = parties;
        this.barrierCommand = barrierAction;
    }

    public CyclicBarrier(int parties) {
        this(parties, null);
    }
```

我就讲一下 CyclicBarrier 中的 `await` 方法，这个类的核心方法

> await

```java
    public int await() throws InterruptedException, BrokenBarrierException {
        try {
            return dowait(false, 0L);
        } catch (TimeoutException toe) {
            throw new Error(toe); // cannot happen
        }
    }

    public int await(long timeout, TimeUnit unit)
        throws InterruptedException,
               BrokenBarrierException,
               TimeoutException {
        return dowait(true, unit.toNanos(timeout));
    }
```

await 方法有两种，一种包含等待超时的，超时会破坏屏障，导致屏障不可用，超时后，若继续使用屏障会抛出 BrokenBarrierException，我们来看一下 doAwait 里面的源码

> doAwait

```java
private int dowait(boolean timed, long nanos)
    throws InterruptedException, BrokenBarrierException,
            TimeoutException {
    // todo 使用可重入锁上锁
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        // todo 记录当前代
        final Generation g = generation;

        // todo 如果屏障被破坏了就抛异常
        if (g.broken)
            throw new BrokenBarrierException();
        // todo 中断了也抛异常
        if (Thread.interrupted()) {
            breakBarrier();
            throw new InterruptedException();
        }

        // todo 当前 count --
        int index = --count;
        // todo 如果 index == 0 说明所有的线程已经准备就绪，需要进行后续处理已经相关唤醒
        if (index == 0) {  // tripped
            boolean ranAction = false;
            try {
                final Runnable command = barrierCommand;
                // todo 调用前置任务
                if (command != null)
                    command.run();
                ranAction = true;
                // todo 分配下一次使用的资源，同时唤醒条件等待任务
                nextGeneration();
                return 0;
            } finally {
                // todo 前置任务失败了，破坏当前屏障
                if (!ranAction)
                    breakBarrier();
            }
        }

        // loop until tripped, broken, interrupted, or timed out
        for (;;) {
            try {
        // todo 阻塞住，知道所有线程准备就绪后进行通知
                if (!timed)
                    trip.await();
                else if (nanos > 0L)
                    nanos = trip.awaitNanos(nanos);
            } catch (InterruptedException ie) {
                if (g == generation && ! g.broken) {
                    breakBarrier();
                    throw ie;
                } else {
                    // We're about to finish waiting even if we had not
                    // been interrupted, so this interrupt is deemed to
                    // "belong" to subsequent execution.
                    Thread.currentThread().interrupt();
                }
            }

            if (g.broken)
                throw new BrokenBarrierException();

            if (g != generation)
                return index;

            // todo 如果超时了，破坏屏障并且抛出异常
            if (timed && nanos <= 0L) {
                breakBarrier();
                throw new TimeoutException();
            }
        }
    } finally {
        lock.unlock();
    }
}
```

> breakBarrier

```java
    private void breakBarrier() {
        generation.broken = true;
        count = parties;
        trip.signalAll();
    }
```

> nextGeneration

```java
    /*
     * todo 唤醒所有条件等待的线程，重置 count 以及 generation 可以进行下一轮使用
     */
    private void nextGeneration() {
        // signal completion of last generation
        trip.signalAll();
        // set up next generation
        count = parties;
        generation = new Generation();
    }
```

> isBroken

```java
    public boolean isBroken() {
        // todo 判断当前屏障是否可用
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return generation.broken;
        } finally {
            lock.unlock();
        }
    }
```

> reset

```java
    public void reset() {
        // todo 重置屏障 （先破坏，在重新建立，防止之前使用的出错，直接使在使用中的抛异常）
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            breakBarrier();   // break the current generation
            nextGeneration(); // start a new generation
        } finally {
            lock.unlock();
        }
    }
```

> getNumberWaiting

```java
    public int getNumberWaiting() {
        // todo 获取在等待的线程数
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return parties - count;
        } finally {
            lock.unlock();
        }
    }
```

最后上一个使用 CyclicBarrier 实现的吃饭的例子帮助理解

```java
public class CyclicBarrierTest {
    public static void main(String[] args) throws InterruptedException {
        // 一共5个人来吃饭
        final CyclicBarrier cyclicBarrier = new CyclicBarrier(5, () -> {
            System.out.println("全部到齐，开始吃饭");
        });

        ExecutorService service = Executors.newCachedThreadPool();

        for (int i = 0; i < 5; i++) {
            final Integer user = i + 1;
            service.execute(() -> {
                try {
                    Thread.sleep((long) (Math.random() * 10000));
                    System.out.println("用户" + user + "在路上");
                    Thread.sleep((long) (Math.random() * 10000));
                    System.out.println("用户" + user + "已到达，等待吃饭，已经到了" + (cyclicBarrier.getNumberWaiting() + 1));
                    cyclicBarrier.await();
                } catch (InterruptedException | BrokenBarrierException e) {
                    e.printStackTrace();
                }

            });
        }
        service.shutdown();
    }
}
```

结果

```java
用户3在路上
用户5在路上
用户3已到达，等待吃饭，已经到了1
用户1在路上
用户1已到达，等待吃饭，已经到了2
用户4在路上
用户5已到达，等待吃饭，已经到了3
用户2在路上
用户2已到达，等待吃饭，已经到了4
用户4已到达，等待吃饭，已经到了5
全部到齐，开始吃饭
```

> 本文首次发布于 [fyypumpkin Blog](http://fyypumpkin.github.io), 作者 [@fyypumpkin](http://github.com/fyypumpkin) ,转载请保留原文链接.