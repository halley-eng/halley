---
title: "并发编程框架 AbstractQueuedSynchronizer 独占和共享锁"
date: 2020-11-25T21:39:51+08:00
draft: true
---



### 独占锁加锁(不响应超时和中断)

通过互斥量state获取锁: 尝试使用CAS更改状态, 
   1. 如果能够获取成功, 则直接获取锁, 退出;
   2. 获取失败, 进入等待队列
        1. addWaiter(Node.EXCLUSIVE)
        2. acquireQueued



```java
    public final void acquire(int arg) {

        /**
         * {@link ReentrantLock.NonfairSync#tryAcquire(int)} 非公平方式获取锁
         *
         * 1. 先尝试(使用CAS)获独占锁;
         * 2. 没有获得独占锁, 将自己作为一个独占节点并加入等待队列,
         *    之后在 acquireQueued 中进入不断等待的时期;
         *
         */
        if (!tryAcquire(arg) &&  // 先尝试获取独占锁, 如果没有申请到锁 才往下走
                // 1. 将当前线程加入到同步队列,并且其占用模式为独占形式
                // 2. 当前线程wait并等待自己的节点的状态
                acquireQueued(addWaiter(Node.EXCLUSIVE), arg))

            selfInterrupt();

    }
```

#### 获取锁


获取锁的函数如下, 其需要子类，根据场景不同(是否公平锁、是否共享锁等)做不同的实现

```java
    protected boolean tryAcquire(int arg) {
        throw new UnsupportedOperationException();
    }

```

然后，不能获取锁会将当前线程封装成为一个等待节点, 并加入到CLH队列

#### 封装当前线程节点并追加到CLH队列

```java
    /**
     * 同步队列中 创建节点并将其加入到CLH队列
     * Creates and enqueues node for current thread and given mode.
     *
     * @param mode Node.EXCLUSIVE for exclusive, Node.SHARED for shared
     * @return the new node
     */
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;               // 暂存旧的尾巴节点   将新创建的节点 链接到 锁等待链表的尾巴节点;
        if (pred != null) { // 证明之前的队列不为空;
            node.prev = pred;           // 新节点链接到前尾巴节点;
            if (compareAndSetTail(pred, node)) {    // 原子的替换当前锁等待链表的尾巴节点, 新加入的节点成为链表的尾巴;
                pred.next = node;
                return node;   // 如果能够成功的将新节点链接到原本的节点, 就可以在这里结束了；
            }
        }
        // 如果当前CLH队列是空的或者是存在并发往链表添加节点的问题 , 将会走到这里, 其会创建一个假的头节点, 并加入进去;
        enq(node);
        return node;
    }

```

其依赖并发append到队列的过程 enq

```java
    /**
     * 保证新加入的节点能够加入到等待队列里面
     * 1. 初始化等待队列;
     * 2. 并发操作链表形式的等待队列;
     *
     * Inserts node into queue, initializing if necessary. See picture above.
     * @param node the node to insert    将要插入的节点;
     * @return node's predecessor        节点的前缀节点(之前的尾巴节点或者是一个空节点);
     */
    private Node enq(final Node node) {
        for (;;) { // 使用乐观锁的方式确保往锁链里面添加数据是线程安全的;
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))  // 初始化头节点
                    tail = head;                    // 使用头节点更新尾巴节点; 这个时候链表的头节点和尾巴节点都为同一个空的链表节点, 其不代表任何线程;
            } else {
                node.prev = t;                      // 当前节点的父亲节点为之前尾巴节点
                if (compareAndSetTail(t, node)) {   // 将node节点升级为尾巴节点
                    t.next = node;                  // 将之前的节点更新为当前节点的前缀
                    return t;
                }
                // 这里有一个并发技巧, 新加入的节点 因为只有当前线程使用 因此不用考虑并发问题, 但是
                // 当当前节点升级为等待链表的尾巴节点的之后, 才修改原本的链表尾巴节点到当前节点的指针;
            }
        }

        /**
         *  并发技巧:
         *
         *  for(;;){
         *      这里面其实可以包含多个分支, 表示外部的请求可以先进入其中一个分支进行初始化的操作
         *      之后，在第二次进入的时候, 根据第一次循环初始化的结果 在进行其他工作;
         *  }
         *
         */
    }

```

#### 阻塞当前线程

如果信号量为条件变量, 那么存在如下循环子过程: 

1. 拿信号量
    1. 失败: 
        1. 询问是否park
            1. 是： park
            2. 否： 继续拿信号量
    2. 成功:
        1. 当前线程节点Node称为头节点;

注意这里退出条件只有, 前缀为头节点并且本次拿到信号量. 


```java
    /**
     * 对于已经在同步队列的节点, 设置同步状态为独占不可中断的等待模式
     *
     * 如果没有获取到锁, 将后调用此方法加入到队列
     *
     * 1. 如果自己的前缀是头节点, 那么尝试获取锁
     *     1.1 如果自己成功获得了锁, 自己将升级为头节点, 然后将前缀节点和后继节点断开;
     *     1.2 如果前缀不是头节点 或者是自己并没有竞争获得锁;
     *
     * 2. 如果前缀不是头节点, 尝试修改自己前缀节点的status, 使得其释放锁后, 去唤醒自己后继
     *    节点, 及相关的线程;
     *
     * Acquires in exclusive uninterruptible mode for thread already in
     * queue. Used by condition wait methods as well as acquire.
     *
     * @param node the node
     * @param arg the acquire argument
     * @return {@code true} if interrupted while waiting 如果在等待的时候被中断 将会返回true;
     */
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();    // 获取前缀节点;
                if (p == head && tryAcquire(arg)) {   // 前缀节点是头节点, 并且成功获得锁     <<!!!当线程被唤醒后会走到这里!!!>>
                    setHead(node);                    // 当前线程升级为头节点;
                    p.next = null;  // help GC
                    failed = false;
                    return interrupted;
                }

                if (shouldParkAfterFailedAcquire(p, node) &&    // 第一次走到这里不会park 里面修改前缀策略为SIGNAL(-1), 下一次将会park;
                    parkAndCheckInterrupt()) // 在这里                                    <<!!!线程一般会在这里阻塞,直到被唤醒!!!>>
                    interrupted = true;  // 标示当前线程是否被中断过;
            }
        } finally {
            // 获取失败则从CLH队列中取消;
            if (failed)
                cancelAcquire(node);
        }
    }

```

循环获取信号量失败后, 决策是否需要park当前线程

```java
    /**
     * Checks and updates status for a node that failed to acquire.
     * Returns true if thread should block. This is the main signal
     * control in all acquire loops.  Requires that pred == node.prev.
     *
     * @param pred node's predecessor holding status
     * @param node the node
     * @return {@code true} if thread should block
     */
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            /*
             * todo： node节点之前设置状态需要一个release信号 将其唤醒, 所以这是我们可以将node节点的线程阻塞; ？？
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;

        if (ws > 0) {       // 确定好前缀节点(不能是那些被取消的节点/线程)
            /*
               前缀节点被取消了;
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);      // 跳过当前节点以前哪些被取消的节点;
            pred.next = node;
        } else {           //  修改前缀节点的状态, 让其释放锁后记得要唤醒自己的后继节点;
            /*
             * 因为默认waitStatus状态为0, 将会走到这里  意思是修改前缀节点的waitStatus, 让其释放锁后 唤醒下面的节点;
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             *
             * 调用方需要确保在parking之前确实是获取不到任何许可
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }

```

如果需要park则park, 后期如果被中断则该函数会返回
```java
    /**
     * Convenience method to park and then check if interrupted
     *
     * @return {@code true} if interrupted
     */
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }

```






### 独占锁加锁(响应超时和中断)

AQS框架依赖tryAcquire接口, 实现了支持超时的获取锁的接口；

```java
    public final boolean tryAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        return tryAcquire(arg) ||
            doAcquireNanos(arg, nanosTimeout);
    }
```


超时获取方法如下:

如果信号量为条件变量, 那么存在如下循环子过程: 

1. 拿信号量
    1. 失败: 
        1. 检测超时，则获取锁失败;
        2. 询问是否park
            1. 是： park
            2. 否： 继续拿信号量
    2. 成功:
        1. 当前线程节点Node称为头节点;
2. 检测中断标志位;

    
循环退出后： 
 如果获取锁失败, 则将该节点从CLH队列取消;

所以退出条件有：
1. 超时;
2. 被中断;
3. 成功获取锁;
    1. 后期释放锁后, 信号量会还回去, 并其他等待线程;


```java
    private boolean doAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (nanosTimeout <= 0L)
            return false;
        final long deadline = System.nanoTime() + nanosTimeout;
        final Node node = addWaiter(Node.EXCLUSIVE);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return true;
                }

                // 检测超时则获取失败;
                nanosTimeout = deadline - System.nanoTime();            // 获取超时时间;
                if (nanosTimeout <= 0L)
                    return false;

                if (shouldParkAfterFailedAcquire(p, node) &&
                    nanosTimeout > spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanosTimeout);  // 暂停指定的事件后 自动唤醒
                if (Thread.interrupted())   // 如果是通过中断唤醒的 将会抛出异常
                    throw new InterruptedException();
            }
        } finally {
            // 获取失败则从CLH队列中取消;
            if (failed)
                cancelAcquire(node);
        }
    }

```



### 独占锁加锁(响应中断)



```java
    public final void acquireInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (!tryAcquire(arg))
            doAcquireInterruptibly(arg);
    }

```


如果信号量为条件变量, 那么存在如下循环子过程: 

1. 拿信号量
    1. 失败: 
        1. 询问是否park
            1. 是： park
                1. 是否被中断
                    1. 是: 抛出异常
            2. 否： 继续拿信号量
    2. 成功:
        1. 当前线程节点Node称为头节点;

循环退出后:
  如果获取锁失败, 则将该节点从CLH队列取消;

所以退出条件有：
2. 被中断;
3. 成功获取锁;
    1. 后期释放锁后, 信号量会还回去, 并其他等待线程;


```java
    /**
     * Acquires in exclusive interruptible mode.
     * @param arg the acquire argument
     */
    private void doAcquireInterruptibly(int arg)
        throws InterruptedException {
        final Node node = addWaiter(Node.EXCLUSIVE);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    throw new InterruptedException();       // 如果线程被中断过那么会抛异常出去;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

```



### 独占锁解锁

该功能仅依赖子类实现tryRelease

```java
    public final boolean release(int arg) {
        // 尝试释放
        if (tryRelease(arg)) {
            // 如果成功释放锁
            Node h = head;
            if (h != null && h.waitStatus != 0)
                // 解锁后继节点; 如果后继节点是空的,那么从尾巴部分找到最前面的节点并进行解锁;
                // 节点进行解锁;
                unparkSuccessor(h);
            return true;
        }
        return false;
    }

```


```java
    /**
     * 如果存在唤醒节点的后继
     * Wakes up node's successor, if one exists.
     *
     * @param node the node
     */
    private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);               //

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        Node s = node.next;
        // 1. s == null 通常是那些直接就能获得锁, 并没有在等待队列等待的那些;
        // 2. s.waitStatus>0 标示不需要被唤醒(已取消)
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev) // 从后往最前面的需要被唤醒的节点(除了被Cancle的节点)
                if (t.waitStatus <= 0) // >0 的时候继续遍历 但是不作为待环形节点;
                    s = t;
        }
        // 找到node的后继节点;
        if (s != null)
            LockSupport.unpark(s.thread);
    }

```


### 共享锁(不响应中断和超时)


```java
  public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)  // 尝试获取共享锁
            doAcquireShared(arg);   // 去获取共享锁;
    }

```

如果信号量为条件变量, 那么存在如下循环子过程: 

1. 拿信号量
    1. 失败: 
        2. 询问是否park
            1. 是： park
            2. 否： 继续拿信号量
    2. 成功:
        1. 当前线程节点Node称为头节点;
        2. 传播唤醒其他节点
2. 检测中断标志位;


```java
    /**
     * 在共享非中断模式下去获取共享锁;
     * Acquires in shared uninterruptible mode.
     * @param arg the acquire argument
     */
    private void doAcquireShared(int arg) {
        // 添加共享等待节点;
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;

        // 用画图的方法来比喻当前节点下面, 每个节点下面都会有一个loop
        // 标示节点相关线程的
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        // 如果当前节点能拿到共享许可, 那么当前线程不会被阻塞, 另外尝试去唤醒其后继节点;
                        // 当前节点拿到锁释放前缀节点的许可证;
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                // 中断被唤醒的地方也是乐观锁获取锁的过程里面, 被唤醒后将会
                //
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

这里和独占锁获取主要区别在于可以级连唤醒其他其他CLH队列中的其他共享节点;

```java
    /**
     * Sets head of queue, and checks if successor may be waiting
     * in shared mode, if so propagating if either propagate > 0 or
     * PROPAGATE status was set.
     *
     * @param node the node
     * @param propagate the return value from a tryAcquireShared
     */
    private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; // Record old head for check below
        setHead(node);
        /*
         * Try to signal next queued node if:
         *   Propagation was indicated by caller,
         *     or was recorded (as h.waitStatus either before
         *     or after setHead) by a previous operation
         *     (note: this uses sign-check of waitStatus because
         *      PROPAGATE status may transition to SIGNAL.)
         * and
         *   The next node is waiting in shared mode,
         *     or we don't know, because it appears null
         *
         * The conservatism in both of these checks may cause
         * unnecessary wake-ups, but only when there are multiple
         * racing acquires/releases, so most need signals now or soon
         * anyway.
         */
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            // 新节点后继为共享节点或者空节点, 则在
            if (s == null || s.isShared())
                doReleaseShared();          // 释放前缀节点的许可证;
        }
    }

```


### 共享锁(响应中断和超时)

原理和独占锁支持中断和超时的方法一致;


### 释放共享锁


```java
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }

```

当前节点状态为SIGNAL, 且修改状态为0的时候, 才会去 unparkSuccessor

```java
    /**
     * 共享模式下释放节点;
     * Release action for shared mode -- signals successor and ensures
     * propagation. (Note: For exclusive mode, release just amounts
     * to calling unparkSuccessor of head if it needs signal.)
     */
    private void doReleaseShared() {
        /*
         * Ensure that a release propagates, even if there are other
         * in-progress acquires/releases.  This proceeds in the usual
         * way of trying to unparkSuccessor of head if it needs
         * signal. But if it does not, status is set to PROPAGATE to
         * ensure that upon release, propagation continues.
         * Additionally, we must loop in case a new node is added
         * while we are doing this. Also, unlike other uses of
         * unparkSuccessor, we need to know if CAS to reset status
         * fails, if so rechecking.
         */
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {   // 检测是否需要唤醒后继节点;
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;          // loop to recheck cases
                    unparkSuccessor(h);    // 唤醒后继节点; 其实唤醒该节点后其会重新获取锁
                }
                // 本次释放操作被其他线程抢先release了, 这个时候需要, 传播release操作;
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;              // loop on failed CAS
            }
            // 在修改状态的时候如果出现添加节点的情况, 需要重新开始释放共享锁;
            if (h == head)                 // loop if head changed
                break;
        }
    }

```



### 总结

如上我们分析了AQS关于独占和共享锁
1. 如何获取和释放锁
2. 如何处理中断和超时

