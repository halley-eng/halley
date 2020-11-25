---
title: "并发编程框架 AbstractQueuedSynchronizer 条件变量"
date: 2020-11-25T21:39:16+08:00
draft: true
---


### wait 去竞争锁

1. 将当前线程加入条件队列
2. 释放当前状态变量 state 个
2. 等待其被转移至竞争队列CLH
3. 等待获取锁, 获取状态变量state个

```java
        /**
         * Implements interruptible condition wait.
         * <ol>
         * <li> If current thread is interrupted, throw InterruptedException.
         * <li> Save lock state returned by {@link #getState}.
         * <li> Invoke {@link #release} with saved state as argument,
         *      throwing IllegalMonitorStateException if it fails.
         * <li> Block until signalled or interrupted. 在被通知或者中断之前阻塞当前线程
         * <li> Reacquire by invoking specialized version of
         *      {@link #acquire} with saved state as argument.
         * <li> If interrupted while blocked in step 4, throw InterruptedException.
         * </ol>
         */
        public final void await() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();           // 将当前线程 加入到等待当前条件队列里面;
            int savedState = fullyRelease(node);        // 释放当前线程曾经申请过到锁;  并尝试线程的当前锁的后继节点
            int interruptMode = 0;
            // 直到当前当前线程的节点 被通知后 移送到同步队列
            while (!isOnSyncQueue(node)) {              // 如果该节点没有被加入到同步队列 将会一直阻塞其线程
                LockSupport.park(this);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            // 节点已经被加入到同步队列
            // 尝试获取锁;
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }

```

### signal 

条件收到唤醒信号后, 通知条件队列中的线程回到CLH竞争队列;

```java
        /**
         * Moves the longest-waiting thread, if one exists, from the
         * wait queue for this condition to the wait queue for the
         * owning lock.
         *
         * @throws IllegalMonitorStateException if {@link #isHeldExclusively}
         *         returns {@code false}
         */
        public final void signal() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignal(first);
        }
```


```java
        /**
         * 直到通知一个没有被取消的节点, 并将其加入到同步队列;
         * Removes and transfers nodes until hit non-cancelled one or
         * null. Split out from signal in part to encourage compilers
         * to inline the case of no waiters.
         * @param first (non-null) the first node on condition queue
         */
        private void doSignal(Node first) {
            do {
                //  将被发送信号节点和它的下线断开, 并让它的下线上位为首位等待节点;
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;
                first.nextWaiter = null;
                // 将一个节点从条件队列转移到主信号队列;
            } while (!transferForSignal(first) &&           // 使用for循环的方式实现递归的效果,避免栈资源的消耗 堪称一绝
                     (first = firstWaiter) != null);
        }

```

将目标线程丢进CLH队列, 并唤醒之;
使得其从前面的await函数中的park部分唤醒, 并得以继续运行至acquireQueued

```java
    /**
     * 将节点从条件队列转移到同步队列;
     *
     * Transfers a node from a condition queue onto sync queue.
     * Returns true if successful.
     * @param node the node  即将转移的节点
     * @return true if successfully transferred (else the node was
     * cancelled before signal)      如果被成功转移将会返回true;
     */
    final boolean transferForSignal(Node node) {
        /*
        *  原子的更新等待者的状态;  从条件队列节点, 转移到同步等待队列;
         * If cannot change waitStatus, the node has been cancelled.
         */
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;

        /*
         *
         * Splice onto queue and try to set waitStatus of predecessor to
         * indicate that thread is (probably) waiting. If cancelled or
         * attempt to set waitStatus fails, wake up to resync (in which
         * case the waitStatus can be transiently and harmlessly wrong).
         */
        Node p = enq(node); // 将节点插入到尾巴, 并返回之前的尾巴节点;    将当前线程从条件队列转移到同步队列  并返回它的前缀节点
        int ws = p.waitStatus;
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))    // 前缀节点已经被取消 或者给前缀节点设置状态失败 将会唤醒节点 todo: 后一种唤醒方式??
            LockSupport.unpark(node.thread); // 解锁node绑定的线程, 解锁后回去重新竞争锁
        return true;
    }

```

### signalAll 

```java
        /**
         * Moves all threads from the wait queue for this condition to
         * the wait queue for the owning lock.
         *
         * @throws IllegalMonitorStateException if {@link #isHeldExclusively}
         *         returns {@code false}
         */
        public final void signalAll() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignalAll(first);
        }
```

```java
        private void doSignalAll(Node first) {
            lastWaiter = firstWaiter = null;
            do {
                Node next = first.nextWaiter;
                first.nextWaiter = null;
                transferForSignal(first);
                first = next;
            } while (first != null);
        }
```



### 总结

1. wait 将当前线程加入到条件队列, 并等待进入同步队列后竞争锁
2. signal 会将条件队列中的一个线程转移到同步队列，使得其有机会竞争锁.

