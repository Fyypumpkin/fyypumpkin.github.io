---
layout:     post
title:      Java ThreadPoolExecutor 实现 之 SynchronousQueue （三）
subtitle:   了解 Java J.U.C 包下的 ThreadPoolExecutor 原理之 SynchronousQueue 
date:       2019-01-08
author:     fyypumpkin
header-img: img/post-bg-digital-native.jpg
catalog: true
tags:
    - Java
    - J.U.C
    - ThreadPool
    - SynchronousQueue
---

## 正文

继上篇博客，本次讲一下 `SynchronousQueue` 同步阻塞队列，粗看 `SynchronousQueue` 代码，会发现并没有像 `LinkedBlockingQueue` 一样，有一个容器的概念，在 `SynchronousQueue` 中，所有的操作都是
 "成对" 出现的，怎么说呢，就是一个生产者生产出来的消息，必须有一个消费者去消费，生产者才能继续生产，否则就会阻塞等待，知道有一个消费者来消费，这样成对的生产消费。
 
 简单看一下用法，我写了一个简单的 demo
 
 > SynchronousQueue 简单用法 （就是 put/take）
 
 ```java
        new Thread(() -> {
             try {
                 System.out.println("producer start put");
                 synchronousQueue.put("1");
             } catch (InterruptedException e) {
                 e.printStackTrace();
             }
         }).start();
 
         Thread.sleep(2000);
         new Thread(() -> {
             try {
                 while (true) {
                     System.out.println(synchronousQueue.take());
                 }
             } catch (InterruptedException e) {
                 e.printStackTrace();
             }
         }).start();
 
         synchronousQueue.put("2");
 ```
 
 结果: 
 
 ```java
 producer start put
 2
 1
 ```
 
 实际上，我感觉很类似最大容量为 1 的普通的队列，但是还是有区别的，主要区别就是在队列空闲的时候，普通队列第一个元素一定能 put 成功并且立刻返回，但是 `synchronousQueue` 和容量无关，只和消费者有关。
 
 为什么要设计一个同步的队列呢？网上也没有很多明确的说法，当然，日常实际上我也没有使用到这个队列，我认为，一个是可以确保消息有消费者消费，发送方能够知道消息已经有人消费了，不过也是我自己 yy 出来的哈哈。
 
 下面来看一下实现吧。
 
 > 构造器
 
 ```java
     public SynchronousQueue() {
         this(false);
     }
 
     public SynchronousQueue(boolean fair) {
         transferer = fair ? new TransferQueue<E>() : new TransferStack<E>();
     }
 ```
 
 从构造器中可以看出，在同步队列内部有公平和非公平的两种选择，默认情况下是非公平的，这次主要讲一下非公平的实现，也就是通过 `TransferStack` 实现的 `SynchronousQueue`。我们先不看 `TransferStack` 是何方神圣，
 直接从队列最简单的方法入手，即 `take` 和 `put` 方法 （`offer` 和 `poll` 方法是非阻塞的，区别不大，就不讲解了）,先从 `take` 方法下手
 
 > take
 
 ```java
     public E take() throws InterruptedException {
     // 核心调用 transferer 的 transfer 方法
         E e = transferer.transfer(null, false, 0);
         if (e != null)
             return e;
         Thread.interrupted();
         throw new InterruptedException();
     }
 ```
 
 实际上 `take` 方法中只是去调用 `transferer` 中的 `transfer` 方法，将任务都交给了 `transferer`，再看 `put` 方法也是惊人的相似
 
 > put
 
 ```java
    public void put(E e) throws InterruptedException {
         if (e == null) throw new NullPointerException();
         // 核心调用 transferer 的 transfer 方法
         if (transferer.transfer(e, false, 0) == null) {
             Thread.interrupted();
             throw new InterruptedException();
         }
     }
 ```
 
 实际上 `take` 和 `put` 方法没有什么好讲的，主要体现一下 `tansferer` 的作用~，下面来主要看一下非公平的实现 `TransferStack`
 
 > TransferStack 变量
 
 ```java
         // 标识生产者
         static final int REQUEST    = 0;
         // 标识消费者
         static final int DATA       = 1;
         // 标识匹配另一个消费者或者生产者
         static final int FULFILLING = 2;
         // 栈顶指针
         volatile SNode head;
 ```
 
 在 `TransferStack` 内部，维护了一个 `SNode`，所有的操作都是基于该数据结构
 
 > SNode
 
 ```java
   static final class SNode {
             // 下一个节点
             volatile SNode next;        // next node in stack
             // 配对的节点
             volatile SNode match;       // the node matched to this
             // 被中断的线程
             volatile Thread waiter;     // to control park/unpark
             Object item;                // data; or null for REQUESTs
             // 当前 node 的模式
             int mode;
             // Note: item and mode fields don't need to be volatile
             // since they are always written before, and read after,
             // other volatile/atomic operations.
             // 翻译一下，这里的 item 和 mode 不需要是 volatile 的，因为有其他 volatile 保证了先写后读
                
             SNode(Object item) {
                 this.item = item;
             }
 
             // cas 替换 next 节点
             boolean casNext(SNode cmp, SNode val) {
                 return cmp == next &&
                     UNSAFE.compareAndSwapObject(this, nextOffset, cmp, val);
             }
 
             boolean tryMatch(SNode s) {
              //todo 通过 cas 操作将节点的 match 设置为相应的 node，设置成功，唤醒相应的 waiter
                 if (match == null &&
                     UNSAFE.compareAndSwapObject(this, matchOffset, null, s)) {
                     Thread w = waiter;
                     if (w != null) {    // waiters need at most one unpark
                         waiter = null;
                         LockSupport.unpark(w);
                     }
                     return true;
                 }
                 // todo 返回 false 可能是存在竞争导致 match 呗其他线程先 set 了
                 return match == s;
             }
 
             /**
              * Tries to cancel a wait by matching node to itself.
              */
             void tryCancel() {
             // 将当前节点的 match 修改为 this
                 UNSAFE.compareAndSwapObject(this, matchOffset, null, this);
             }
 
             boolean isCancelled() {
                 return match == this;
             }
 
             // Unsafe 类的一些初始化操作
             // Unsafe mechanics
             private static final sun.misc.Unsafe UNSAFE;
             private static final long matchOffset;
             private static final long nextOffset;
 
             static {
                 try {
                     UNSAFE = sun.misc.Unsafe.getUnsafe();
                     Class<?> k = SNode.class;
                     matchOffset = UNSAFE.objectFieldOffset
                         (k.getDeclaredField("match"));
                     nextOffset = UNSAFE.objectFieldOffset
                         (k.getDeclaredField("next"));
                 } catch (Exception e) {
                     throw new Error(e);
                 }
             }
         }
 ```
 
 SNode 里面包含比较重要的方法就是 `tryMatch`，`tryMatch` 方法会尝试将当前的 SNode 和入参的 Node 进行一个配对，配对成功就释放 SNode 中等待的线程，
 失败就返回 `false`，节点中的成员变量 `match` 存放的就是配对的节点，`match` 等于 `null` 就说明没有配对过，`match == this`，就说明这个节点已经被取消了
 
 > awaitFulfill
 
 ```java
 SNode awaitFulfill(SNode s, boolean timed, long nanos) {
          
             final long deadline = timed ? System.nanoTime() + nanos : 0L;
             Thread w = Thread.currentThread();
             int spins = (shouldSpin(s) ?
                          (timed ? maxTimedSpins : maxUntimedSpins) : 0);
             for (;;) {
                 // todo 尝试取消，取消操作就是讲 match 设置为当前的对象 s.match == s 表明取消了
                 if (w.isInterrupted())
                     s.tryCancel();
                 SNode m = s.match;
                 if (m != null)
                     return m;
                 if (timed) {
                     nanos = deadline - System.nanoTime();
                     if (nanos <= 0L) {
                         s.tryCancel();
                         continue;
                     }
                 }
                 // todo 自旋
                 if (spins > 0)
                     spins = shouldSpin(s) ? (spins-1) : 0;
                 else if (s.waiter == null)
                     // todo 自旋没找到 match 就设置 waiter 为当前线程，准备阻塞 （还有一次机会）
                     s.waiter = w; // establish waiter so can park next iter
                 else if (!timed)
                     // todo 阻塞当前线程
                     LockSupport.park(this);
                 else if (nanos > spinForTimeoutThreshold)
                     LockSupport.parkNanos(this, nanos);
             }
         }
 ```
 
 可以看到，在 `awaitFulfill` 方法中，存在堵塞等待，在阻塞当前线程前，会有自旋的机会，当自旋次数用完后，就会将 SNode 中的 `waiter` 字段设置成当前线程，
 在下一次循环还没成功的找到匹配，那么就会调用 `LockSupport.park()` 方法 阻塞当前线程，直到别的线程调用 `tryMatch` 方法是配上后在调用 `LockSupport.unpark()`
 方法唤醒线程，如果等待匹配过程中，线程中断了，就会将当前节点的 `match` 设置成当前节点，即 `s.match == this = true`。
 
 下面是重头戏，即 `Stack` 中的 `transfer` 方法。
 
 > transfer
 
 ```java
         E transfer(E e, boolean timed, long nanos) {
             SNode s = null; // constructed/reused as needed
             //  标识是消费者还是生产者 （可以确定是 put 还是 take）
             int mode = (e == null) ? REQUEST : DATA;
 
             for (;;) {
                 SNode h = head;
                 if (h == null || h.mode == mode) {  // empty or same-mode
                     //  设置了 timed 并且时间小于等于 0 标识需要立刻执行
                     if (timed && nanos <= 0) {      // can't wait
                         //  如果是非阻塞的，那么就直接返回 null
                         if (h != null && h.isCancelled())
                             casHead(h, h.next);     // pop cancelled node
                         else
                             return null;
                         //  生成一个新节点，将该节点设置为 head，将原来的 head 设置为该节点的后继 (压栈)
                     } else if (casHead(h, s = snode(s, e, h, mode))) {
                         //  尝试等待配对
                         SNode m = awaitFulfill(s, timed, nanos);
                         //  m == s 一定是被中断了 ==> s.match == s
                         if (m == s) {               // wait was cancelled
                             clean(s);
                             return null;
                         }
                         //  设置头节点，如果当前是头结点的下一个节点
                         if ((h = head) != null && h.next == s)
                             casHead(h, s.next);     // help s's fulfiller
                         //  返回相应的结果 （设置消费方和提供方，都应该会返回）
                         return (E) ((mode == REQUEST) ? m.item : s.item);
                     }
                     //  检查栈顶是否已经配对了，没配对进入进行配对
                 } else if (!isFulfilling(h.mode)) { // try to fulfill
                     if (h.isCancelled())            // already cancelled
                         casHead(h, h.next);         // pop and retry
                     //  入栈
                     else if (casHead(h, s=snode(s, e, h, FULFILLING|mode))) {
                         for (;;) { // loop until matched or waiters disappear
                             //  获取当前节点的下一个节点
                             SNode m = s.next;       // m is s's match
                             //  如果next是null，说明当前在栈顶，就将栈顶设置为null，重新新的循环
                             if (m == null) {        // all waiters are gone
                                 casHead(s, null);   // pop fulfill node
                                 s = null;           // use new node next time
                                 break;              // restart main loop
                             }
                             //  记录后续，方便cas
                             SNode mn = m.next;
                             //  尝试配对
                             if (m.tryMatch(s)) {
                                 casHead(s, mn);     // pop both s and m
                                 //  返回相应的结果 （设置消费方和提供方，都应该会返回）
                                 return (E) ((mode == REQUEST) ? m.item : s.item);
                             } else                  // lost match
                             //  其他线程率先设置了 match，就将 m 移出栈
                                 s.casNext(m, mn);   // help unlink
                         }
                     }
                     //  头结点正在匹配
                 } else {                             // help a fulfiller
                     SNode m = h.next;               // m is h's match
                     if (m == null)                  // waiter is gone
                         //  需要弹出头结点
                         casHead(h, null);           // pop fulfilling node
                     else {
                         SNode mn = m.next;
                         if (m.tryMatch(h))          // help match
                             casHead(h, mn);         // pop both h and m
                         else                        // lost match
                             h.casNext(m, mn);       // help unlink
                     }
                 }
             }
         }
```

`transfer` 是该队列的核心方法，因为整体都是基于 `cas` 操作，所以代码阅读起来会比较困难，不是很好理解，我把大部分带么都解释了一遍，如果有不懂的地方，可以在下面留言，我尽量会解答，
相比较而言，该方法的注释已经比较完整了，这次的博客就这么水过去了~，因为队列其他方法都类似的，主要讲一下核心方法和不一样的地方~


> 本文首次发布于 [fyypumpkin Blog](http://fyypumpkin.github.io), 作者 [@fyypumpkin](http://github.com/fyypumpkin) ,转载请保留原文链接.