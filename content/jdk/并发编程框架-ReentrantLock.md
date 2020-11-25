---
title: "并发编程框架 ReentrantLock"
date: 2020-11-25T21:36:05+08:00
draft: true
---


### 同步器


```java
    /**
     * Base of synchronization control for this lock. Subclassed
     * into fair and nonfair versions below. Uses AQS state to
     * represent the number of holds on the lock.
     */
    abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = -5179523762034025860L;

        /**
         * 锁定操作
         *
         * Performs {@link Lock#lock}. The main reason for subclassing
         * is to allow fast path for nonfair version.
         */
        abstract void lock();

        /**
         * 1. 锁空闲:  尝试原子方式将锁状态推到忙碌的状态, 并占用锁
         * 2. 锁忙碌:  检测锁的占用线程是否和当前线程一致;
         *
         * Performs non-fair tryLock.  tryAcquire is implemented in
         * subclasses, but both need nonfair try for trylock method.
         */
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {   // 当前锁是空闲状态
                if (compareAndSetState(0, acquires)) { // 尝试占用锁, 乐观锁的方式获得锁      state是锁实现核心的数据结构, 所有对锁的申请和释放都需要原子的操作该值;
                    setExclusiveOwnerThread(current);          // 将当前线程设置成独占状态;
                    return true;
                }
            }   // 锁忙碌的情况下
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;   // 增加当前线程锁的次数
                if (nextc < 0) // overflow  超过整数的范围
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);    // 当前线程已经获得了锁, 可以随意改变当前的锁定状态, 不存在线程安全的问题,  因为当前过程只有锁的持有线程才能够调用;
                return true;    // 成功获得了锁;
            }
            return false;
        }

        protected final boolean tryRelease(int releases) {
            int c = getState() - releases; // 剩余锁定次数
            if (Thread.currentThread() != getExclusiveOwnerThread()) // 如果不是在锁绑定的线程里面执行此方法 将会抛出异常
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {       // 当且仅仅当锁的申请次数正好等于本次释放次数时 才能将锁释放;
                free = true;    // 代表是否成功将锁释放;
                setExclusiveOwnerThread(null); // 将当前线程从锁中解绑
            }
            setState(c);        // 设定剩余锁定次数
            return free;
        }

        protected final boolean isHeldExclusively() {
            // While we must in general read state before owner,
            // we don't need to do so to check if current thread is owner
            return getExclusiveOwnerThread() == Thread.currentThread();
        }

        final ConditionObject newCondition() {
            return new ConditionObject();
        }

        // Methods relayed from outer class

        final Thread getOwner() {
            return getState() == 0 ? null : getExclusiveOwnerThread();
        }

        final int getHoldCount() {
            return isHeldExclusively() ? getState() : 0;
        }

        final boolean isLocked() {
            return getState() != 0;
        }

        /**
         * Reconstitutes the instance from a stream (that is, deserializes it).
         */
        private void readObject(java.io.ObjectInputStream s)
            throws java.io.IOException, ClassNotFoundException {
            s.defaultReadObject();
            setState(0); // reset to unlocked state
        }
    }
```


#### 非公平锁

```java
    /**
     * 非公平锁
     * 首先会通过乐观锁的方式去竞争锁, 竞争失败才去排队
     * Sync object for non-fair locks
     */
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        /**
         * Performs lock.  Try immediate barge, backing up to normal
         * acquire on failure.
         */
        final void lock() {
            // 首先尝试使用乐观锁 以原子更新的方式获得锁
            if (compareAndSetState(0, 1))   // 通过乐观锁的方式去竞争锁
                // 独占锁;
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);                           // 竞争不到才去排队;
            /**
             * 获取锁的流程
             * 1. {@link ReentrantLock} cas 直接获取锁
             * 2. 转发给AQS
             *    {@link AbstractQueuedSynchronizer#acquire(int)}
             *    2.1 tryAcquire:尝试通过非公平竞争的方式获取锁;  最终使用 {@link NonfairSync#tryAcquire(int)}
             *      如果竞争不到加入到等待队列中;
             *    2.2 addWaiter
             *      在当前线程尝试修改前缀的状态, 让其在释放锁后 通知当前线程;
             *
             *
             * {@link NonfairSync#tryAcquire(int)}
             *      {@link Sync#nonfairTryAcquire(int)}  尝试非公平的方式获取锁
             *
             * {@link AbstractQueuedSynchronizer#addWaiter(java.util.concurrent.locks.AbstractQueuedSynchronizer.Node)}
             */
        }

        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    }
```


#### 公平锁

```java
    /**
     * 公平锁
     * 直接去排队;
     * Sync object for fair locks
     */
    static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;

        final void lock() {
            acquire(1);
        }

        /**
         * Fair version of tryAcquire.  Don't grant access unless
         * recursive call or no waiters or is first.
         */
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
    }
```


### 总结

ReentrantLock 是 AQS的具体实现

两种情况能够拿到锁(nonfairTryAcquire 返回true)
1. 当前state == 0, cas 为 1；
2. 当前state !=0 但是独占线程为当前线程, cas 当前状态为 oldState + 1;


一种情况能够成功释放锁(tryRelease 返回true)
1. 释放信号量后state为0;







