---
layout:     post
title:      Java ReentrantLock AQS 实现(非公平)（二）
subtitle:   了解 ReentrantLock 中 Condition 的原理
date:       2018-12-12
author:     fyypumpkin
header-img: img//post-bg-debug.png
catalog: true
tags:
    - Java
    - AQS
    - J.U.C
    - ReentrantLock
    - Condition
---

## 正文
继上一篇博客，这一片博客讲一下 `Condition` 中的 `signal` 以及 `await` 这两个方法的原理，为了解决 `lock` 和 `unlock` 之间能够实现 `Object` 的 `wait` 和 `notify` 的效果， `ReentrantLock` 中引入了 `Condition` 的概念。

### 基本的 `Condition` 用法

> 贴一段经典的消费者生产者的 Condition 实现

```java
public class ReentrantLockConsumer {
    private static Integer count = 0;
    private static final Integer FULL = 10;
    //创建一个锁对象
    private Lock lock = new ReentrantLock();
    //创建两个条件变量，一个为缓冲区非满，一个为缓冲区非空
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();
    public static void main(String[] args) {
        ReentrantLockConsumer test2 = new ReentrantLockConsumer();
        new Thread(test2.new Producer()).start();
        new Thread(test2.new Consumer()).start();
    }
    class Producer implements Runnable {
        @Override
        public void run() {
            for (int i = 0; i < 10; i++) {
                try {
                    Thread.sleep(1000);
                } catch (Exception e) {
                    e.printStackTrace();
                }
                //获取锁
                lock.lock();
                lock.lock();
                try {
                    while (count == FULL) {
                        try {
                            notFull.await();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                    count++;
                    System.out.println(Thread.currentThread().getName()
                            + "生产者生产，目前总共有" + count);
                    //唤醒消费者
                    notEmpty.signal();
                } finally {
                    //释放锁
                    lock.unlock();
                    lock.unlock();
                }
            }
        }
    }
    class Consumer implements Runnable {
        @Override
        public void run() {
            for (int i = 0; i < 10; i++) {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e1) {
                    e1.printStackTrace();
                }
                lock.lock();
                lock.lock();
                try {
                    while (count == 0) {
                        try {
                            notEmpty.await();
                        } catch (Exception e) {
                            e.printStackTrace();
                        }
                    }
                    count--;
                    System.out.println(Thread.currentThread().getName()
                            + "消费者消费，目前总共有" + count);
                    notFull.signal();
                } finally {
                    lock.unlock();
                    lock.unlock();
                }
            }
        }
    }
}
``` 

代码比较长，实现的就是最简单的生产者和消费者，可以看到我分配了两个对象，一个用于标识当前池子中是否为空，一个判断池子是否为满的，当池子为空时，那么消费者应该需要等待生产者进行生产后才能继续消费，这个逻辑通过 `notRmpty.await()` 实现
而生产者这边每次生产都会判断当前池子是否满了，满了就等待消费者消费完在继续生产，通过 ` notFull.await();` 实现，当消费者消费了之后，就通知生产者生产，通过 ` notFull.signal()` 实现，生产者生产了就通知消费者消费，通过 `notEmpty.signal();`
实现，实际上，整个过程就是在协调两边何时消费何时生产，这个简单的例子主要用来说明 `Condition` 如何使用。

> 看一下 Condition 对象中所包含的成员

```java
  /** First node of condition queue. */
        private transient Node firstWaiter;
        /** Last node of condition queue. */
        private transient Node lastWaiter;
```

很容易联想到上个博客里面说的 AQS 中的队列，同样包含一个 head 和一个 tail，而 `Condition` 中则叫做 `firstWaiter` 和 `lastWaiter`,所以可以猜想，`Condition` 中同样是维护着和 AQS 中类似的一个队列，我们继续向下看

> Condition 中的 await 方法

```java
        public final void await() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            // 使用当前线程构造一个 Node，并加到队尾
            Node node = addConditionWaiter();
            // 需要考虑重入的情况，会释放所有重入的锁
            int savedState = fullyRelease(node);
            int interruptMode = 0;

            // 判断当前线程是否在等待队列中，如果不在，就阻塞等待
            while (!isOnSyncQueue(node)) {
                LockSupport.park(this);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }
```

```java
        // 将新建的 node 加入到队尾
        private Node addConditionWaiter() {
            Node t = lastWaiter;
            // If lastWaiter is cancelled, clean out.
            // 只可能是CANCELLED和CONDITION
            if (t != null && t.waitStatus != Node.CONDITION) {
                unlinkCancelledWaiters();
                t = lastWaiter;
            }
            Node node = new Node(Thread.currentThread(), Node.CONDITION);
            if (t == null)
                firstWaiter = node;
            else
                t.nextWaiter = node;
            lastWaiter = node;
            return node;
        }
```

```java
        // 清楚Condition队列中状态时CANCELLED的（Condition队列中应该只会有两种状态， condition和cancelled）
        private void unlinkCancelledWaiters() {
            Node t = firstWaiter;
            Node trail = null;
            while (t != null) {
                Node next = t.nextWaiter;
                if (t.waitStatus != Node.CONDITION) {
                    t.nextWaiter = null;
                    if (trail == null)
                        firstWaiter = next;
                    else
                        trail.nextWaiter = next;
                    if (next == null)
                        lastWaiter = trail;
                }
                else
                    trail = t;
                t = next;
            }
        }
```

```java
  final int fullyRelease(Node node) {
        boolean failed = true;
        try {
            int savedState = getState();
            if (release(savedState)) {
                failed = false;
                return savedState;
            } else {
                throw new IllegalMonitorStateException();
            }
        } finally {
            if (failed)
                node.waitStatus = Node.CANCELLED;
        }
    }
```

上面四个方法比较简单，我就直接放在一块说了，调用 `await` 方法首先会判断当前是否是当前线程独占的，不是的话就直接抛异常了，否则就进入后续的判断，
首先会使用当前线程构造一个 node，然后将这个 node 加入到队尾，加入队尾的时候，如果 `lastWaiter` 状态不是 `CONDITION` ，那么就会执行清楚所有状态为非 `CONDITION` 的节点去除，
然后将使用当前线程构造的 node 放入到队尾并返回，返回后开始执行锁的释放 `fullyRelease` 方法，考虑到锁的重入，会直接获取当前锁进入的次数 `state`，然后执行 `release` 将整个锁释放掉，
并返回锁的重入次数。

接下来，调用 `isOnSyncQueue` 方法，判断当前的 node 在AQS是否包含，这里，两个队列就互相关联了，我们看一下 `isOnSyncQueue` 怎么写的

```java
    final boolean isOnSyncQueue(Node node) {
        // 如果当前状态是CONDITION或者前驱是null的表示不再AQS上，如果有后继节点，那么一定在AQS上，否则就去遍历
        if (node.waitStatus == Node.CONDITION || node.prev == null)
            return false;
        if (node.next != null) // If has successor, it must be on queue
            return true;
        return findNodeFromTail(node);
    }
    private boolean findNodeFromTail(Node node) {
        Node t = tail;
        for (;;) {
            if (t == node)
                return true;
            if (t == null)
                return false;
            t = t.prev;
        }
    }    
```

`isOnSyncQueue` 中，会先判断 node 的状态，如果 node 的状态还是 `CONDITION` 或者前驱是 `null`，那么就说明没有进入过 AQS，就直接 return false，
如果 node 已经有后继节点，那么说明一定进入过 AQS，否则就直接通过队列遍历来进行查找，然后返回结果，如果不在 AQS 队列中，就阻塞等待，否则就进入 `acquireQueued`
方法来尝试竞争锁（这里会走 lock 方法一样的流程），然后会 `unlinkCancelledWaiters` 清理不是条件等待的节点。

实际上 `await` 的思想很简单，大概思想为：首先将此代表该当前线程的节点加入到条件队列中去，然后释放该线程所有的锁并开始睡眠，最后不停的检测 AQS 队列中是否出现了此线程节点。
如果收到 signal 信号之后就会在 AQS 队列中检测到，检测到之后，说明此线程又参与了竞争锁。

下面来说一下 `signal` 方法

> signal

```java
        public final void signal() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignal(first);
        }
```

和 `await` 方法一样， 首先会判断当前是否是当前线程独占的，不是的话就直接抛异常了，然后获取条件等待队列的第一个节点 `firstNode`， 如果节点不为空，那么就执行 `doSignal`
方法。

> doSignal

```java
        private void doSignal(Node first) {
            do {
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;
                first.nextWaiter = null;
            } while (!transferForSignal(first) &&
                     (first = firstWaiter) != null);
        }

```

`doSignal` 方法中是一个 do-while 循环, 这个循环会不断检查 `transferForSignal` 的状态和 `firstWaiter` 是否为 null，看一下 `transferForSignal` 代码。

> transferForSignal

```java
    final boolean transferForSignal(Node node) {
        /*
         * If cannot change waitStatus, the node has been cancelled.
         */
        // 这里会做值的变更，将CONDITION状态变更为0，变更成功后即将放入AQS队列
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;
        Node p = enq(node);
        int ws = p.waitStatus;
        //如果结点p的状态为cancel 或者修改waitStatus失败，则直接唤醒,正常情况下应该是不会唤醒的
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            LockSupport.unpark(node.thread);
        return true;
    }
```

从 `transferForSignal` 代码中可以看到，首先会直接进行一个值变更，将当前 node 的状态由条件等待设置为 0，如果失败那么久返回 false， 则会重新进入外面的 do-while (并发下可能被别人先唤醒了，firstWaiter 可能为 null 而退出循环)
直到更新成功，之后便调用 `enq` 方法，将 node 节点加入到 AQS 队列中，然后返回 node，检查 node 的状态，如果结点的状态为cancel 或者修改waitStatus失败，则直接唤醒,正常情况下应该是不会唤醒的，就返回 true，使外面的循环终止，将整个过程就交给了
AQS 队列来维护了

引用另外一篇 [博客](https://blog.csdn.net/u010412719/article/details/52089561) 的例子

```java
 public class ConditionDemo {
        private static Lock lock = new ReentrantLock();
        private static Condition condition = lock.newCondition();
        public static void main(String[] args) {
            Thread thread1 = new Thread(new Runnable(){

                @Override
                public void run() {
                    lock.lock();    
                    System.out.println(Thread.currentThread().getName()+"正在运行。。。。");
                    try {
                        Thread.sleep(2000);
                        System.out.println(Thread.currentThread().getName()+"停止运行，等待一个signal");
                        condition.await();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.println(Thread.currentThread().getName()+"获得一个signal，继续执行");
                    lock.unlock();
                }

            },"waitThread");
            thread1.start();

            try {
                Thread.sleep(1000);//保证线程1先执行，否则线程1将一直等待signal信号
            } catch (InterruptedException e1) {
                e1.printStackTrace();
            }
            Thread thread2 = new Thread(new Runnable(){

                @Override
                public void run() {
                    lock.lock();    
                    System.out.println(Thread.currentThread().getName()+"正在运行。。。。");
                    condition.signal();//发送信号，唤醒其它线程
                    System.out.println(Thread.currentThread().getName()+"发送一个signal");
                    System.out.println(Thread.currentThread().getName()+"发送一个signal后，结束");
                    lock.unlock();
                }

            },"signalThread");
            thread2.start();

        }
}
```

结果

```java
waitThread正在运行。。。。
waitThread停止运行，等待一个signal
signalThread正在运行。。。。
signalThread发送一个signal
signalThread发送一个signal后，结束
waitThread获得一个signal，继续执行
```

1、首先，线程1调用lock.lock()时，由于此时锁并没有被其它线程占用，因此线程1直接获得锁并不会进入AQS同步队列中进行等待。

2、在线程1执行期间，线程2调用lock.lock()时由于锁已经被线程1占用，因此，线程2进入AQS同步队列中进行等待。

3、在线程1中执行condition.await()方法后，线程1释放锁并进入条件队列Condition中等待signal信号的到来。

4、线程2，因为线程1释放锁的关系，会唤醒AQS队列中的头结点，所以线程2会获取到锁。

5、线程2调用signal方法，这个时候Condition的等待队列中只有线程1一个节点，于是它被取出来，并被加入到AQS的等待队列中。注意，这个时候，线程1 并没有被唤醒。只是加入到了AQS等待队列中去了

6、待线程2执行完成之后并调用lock.unlock()释放锁之后，会唤醒此时在AQS队列中的头结点.所以线程1开始争夺锁(由于此时只有线程1在AQS队列中，因此没人与其争夺),如果获得锁继续执行。


> 本文首次发布于 [fyypumpkin Blog](http://fyypumpkin.github.io), 作者 [@fyypumpkin](http://github.com/fyypumpkin) ,转载请保留原文链接.