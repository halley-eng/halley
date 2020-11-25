---
title: "并发编程框架 ReentrantReadWriteLock"
date: 2020-11-25T21:35:01+08:00
draft: true
---


## 构造

读写锁都是对 ReentrantReadWriteLock 中的同步器sync的封装，会共享其中的state变量


```java

    /**
     * Creates a new {@code ReentrantReadWriteLock} with
     * default (nonfair) ordering properties.
     */
    public ReentrantReadWriteLock() {
        this(false);
    }

    /**
     * Creates a new {@code ReentrantReadWriteLock} with
     * the given fairness policy.
     *
     * @param fair {@code true} if this lock should use a fair ordering policy
     */
    public ReentrantReadWriteLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
        readerLock = new ReadLock(this);
        writerLock = new WriteLock(this);
    }

    public ReentrantReadWriteLock.WriteLock writeLock() { return writerLock; }
    public ReentrantReadWriteLock.ReadLock  readLock()  { return readerLock; }

```

## 数据结构

读写锁使用
ReetrantReadWriteLock#Sync

```java
        /*
         * Read vs write count extraction constants and functions.
         * Lock state is logically divided into two unsigned shorts:
         * The lower one representing the exclusive (writer) lock hold count,
         * and the upper the shared (reader) hold count.
         */

        static final int SHARED_SHIFT   = 16;
        static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
        static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
        static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

        /** Returns the number of shared holds represented in count  */
        static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
        /** Returns the number of exclusive holds represented in count  */
        static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }

```

## 读锁
![db280648408f6912e156232800e6d199.png](evernotecid://0C0C6CA7-E0B1-4D07-A08B-2457E22E1166/appyinxiangcom/2181761/ENResource/p510)

### 读锁获取

ReetrantReadWriteLock 对外提供如下方法, 以此提供读锁的访问; 
```java
public ReentrantReadWriteLock.ReadLock  readLock()  { 
    return readerLock; 
}
```
起始该读锁的各个方法实现也是基于 ReentrantReadWriteLock
而且写锁也是基于同一个ReentrantReadWriteLock,
本质是都基于它们内部同一个state变量;

所以读锁的加锁过程为

ReadLock#lock

```java
   public void lock() {
            sync.acquireShared(1);
   }
```

AQS#acquireShared
```java
  public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)  // 尝试获取共享锁
            doAcquireShared(arg);   // 去获取共享锁;
    }
```

ReetrantReadWriteLock#tryAcquireShared

```java
        protected final int tryAcquireShared(int unused) {
            /*
             * Walkthrough:
             * 1. If write lock held by another thread, fail.
             * 2. Otherwise, this thread is eligible for
             *    lock wrt state, so ask if it should block
             *    because of queue policy. If not, try
             *    to grant by CASing state and updating count.
             *    Note that step does not check for reentrant
             *    acquires, which is postponed to full version
             *    to avoid having to check hold count in
             *    the more typical non-reentrant case.
             * 3. If step 2 fails either because thread
             *    apparently not eligible or CAS fails or count
             *    saturated, chain to version with full retry loop.
             */
            Thread current = Thread.currentThread();
            int c = getState();
            // 已经被独占并且不是当前线程, 则不能拿到读锁;
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                return -1;
            // 获取读锁获取数量;
            int r = sharedCount(c);


            if (!readerShouldBlock() &&
                r < MAX_COUNT &&
                compareAndSetState(c, c + SHARED_UNIT)) {
                // 第一次读锁初始化;
                if (r == 0) {
                    firstReader = current;
                    firstReaderHoldCount = 1;
                // 累加读锁数目;
                } else if (firstReader == current) {
                    firstReaderHoldCount++;
                // 不是第一次获取读锁, 且不是第一次获取的线程;
                } else {
                    // 获取当前线程的HoldCounter
                    HoldCounter rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current))
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    // 累计当前线程的HoldCounter
                    rh.count++;
                }
                return 1;
            }
            return fullTryAcquireShared(current);
        }
```


```java

        /**
         * Full version of acquire for reads, that handles CAS misses
         * and reentrant reads not dealt with in tryAcquireShared.
         */
        final int fullTryAcquireShared(Thread current) {
            /*
             * This code is in part redundant with that in
             * tryAcquireShared but is simpler overall by not
             * complicating tryAcquireShared with interactions between
             * retries and lazily reading hold counts.
             */
            HoldCounter rh = null;
            for (;;) {
                int c = getState();
                // 被独占并且不是自己, 加锁失败;
                if (exclusiveCount(c) != 0) {
                    if (getExclusiveOwnerThread() != current)
                        return -1;
                    // else we hold the exclusive lock; blocking here
                    // would cause deadlock.
                // 需要阻塞读者, 公平锁: 队列中有其他节点在等待 , 非公平锁, 头节点的后继尾独占锁;
                } else if (readerShouldBlock()) {
                    // Make sure we're not acquiring read lock reentrantly
                    if (firstReader == current) {
                        // assert firstReaderHoldCount > 0;
                    } else {
                        if (rh == null) {
                            rh = cachedHoldCounter;
                            // 已经判定阻塞的情况下, 又有其他线程出现;
                            if (rh == null || rh.tid != getThreadId(current)) {
                                rh = readHolds.get();
                                if (rh.count == 0)
                                    readHolds.remove();
                            }
                        }
                        // 不可能拿到读锁, 直接去阻塞;
                        if (rh.count == 0)
                            return -1;
                    }
                }

                //
                if (sharedCount(c) == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");

                // 修改状态成功, 则获取共享锁, 调用放就可以不阻塞了; 
                if (compareAndSetState(c, c + SHARED_UNIT)) {
                    if (sharedCount(c) == 0) {
                        firstReader = current;
                        firstReaderHoldCount = 1;
                    } else if (firstReader == current) {
                        firstReaderHoldCount++;
                    } else {
                        if (rh == null)
                            rh = cachedHoldCounter;
                        if (rh == null || rh.tid != getThreadId(current))
                            rh = readHolds.get();
                        else if (rh.count == 0)
                            readHolds.set(rh);
                        rh.count++;
                        cachedHoldCounter = rh; // cache for release
                    }
                    return 1;
                }
            }
        }
```
当返回 > 1 式, 请求的读线程就可以不阻塞了;

否则需要调用AQS的 doAcquireShared;

### 




```java
        /**
         * 可以保证当不存在写锁的时候一定能够成功;
         * 1. 与独占锁(写锁)冲突, 则返回false
         * 2. 不冲突情况下:
         *      1. 共享锁数量超过阈值抛出异常；
         *      2. 累计当前共享锁数量, 每个线程的读锁获取次数存放在自己线程上下文中; 
         *
         * Performs tryLock for read, enabling barging in both modes.
         * This is identical in effect to tryAcquireShared except for
         * lack of calls to readerShouldBlock.
         */
        final boolean tryReadLock() {
            Thread current = Thread.currentThread();
            for (;;) {
                int c = getState();
                // 已经被独占并且不是当前线程, 则不能拿到读锁;
                if (exclusiveCount(c) != 0 &&
                    getExclusiveOwnerThread() != current)
                    return false;

                // 获取读锁获取数量;
                int r = sharedCount(c);
                if (r == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");

                // 读锁区域 + 1
                if (compareAndSetState(c, c + SHARED_UNIT)) {
                    // 第一次读锁初始化;
                    if (r == 0) {
                        firstReader = current;
                        firstReaderHoldCount = 1;
                    // 累加读锁数目;
                    } else if (firstReader == current) {
                        firstReaderHoldCount++;
                    // 不是第一次获取读锁, 且不是第一次获取的线程;
                    } else {
                        // 获取当前线程的HoldCounter
                        HoldCounter rh = cachedHoldCounter;
                        if (rh == null || rh.tid != getThreadId(current))
                            cachedHoldCounter = rh = readHolds.get();
                        else if (rh.count == 0)
                            readHolds.set(rh);
                        // 累计当前线程的HoldCounter
                        rh.count++;
                    }

                    //
                    return true;
                }
            }
        }
```




### 读锁释放

```java
  /**
         * Attempts to release this lock.
         *
         * <p>If the number of readers is now zero then the lock
         * is made available for write lock attempts.
         */
        public void unlock() {
            sync.releaseShared(1);
        }

```


### 写锁加锁

WriteLock#lock
```java
  public void lock() {
            sync.acquire(1);
  }
```
AQS#acquire

```java
    /**
     * Acquires in exclusive mode, ignoring interrupts.  Implemented
     * by invoking at least once {@link #tryAcquire},
     * returning on success.  Otherwise the thread is queued, possibly
     * repeatedly blocking and unblocking, invoking {@link
     * #tryAcquire} until success.  This method can be used
     * to implement method {@link Lock#lock}.
     *
     * @param arg the acquire argument.  This value is conveyed to
     *        {@link #tryAcquire} but is otherwise uninterpreted and
     *        can represent anything you like.
     */
    public final void acquire(int arg) {

        /**
         * {@link ReentrantLock.NonfairSync#tryAcquire(int)} 非公平方式获取锁
         *
         * 1. 先尝试获独占锁;
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

ReetrantReadWriteLock#Sync#tryAcquire

```java
        protected final boolean tryAcquire(int acquires) {
            /*
             * Walkthrough:
             * 1. If read count nonzero or write count nonzero
             *    and owner is a different thread, fail.
             * 2. If count would saturate, fail. (This can only
             *    happen if count is already nonzero.)
             * 3. Otherwise, this thread is eligible for lock if
             *    it is either a reentrant acquire or
             *    queue policy allows it. If so, update state
             *    and set owner.
             */
            Thread current = Thread.currentThread();
            int c = getState();
            // 写锁数量;
            int w = exclusiveCount(c);
            if (c != 0) {
                // 有共享锁, 或者是独占线程不是自己, 则写锁改状态失败;
                // (Note: if c != 0 and w == 0 then shared count != 0)
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;
                // 不能获取过量的写锁;
                if (w + exclusiveCount(acquires) > MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                // 设置写锁;
                // Reentrant acquire
                setState(c + acquires);
                return true;
            }
            // 公平锁: CLH队列是否为空, 非公平锁: false
            if (writerShouldBlock() ||
                !compareAndSetState(c, c + acquires))
                return false;
            // 设置独占线程;
            setExclusiveOwnerThread(current);
            return true;
        }
```


### 写锁释放


        /**
         * Attempts to release this lock.
         *
         * <p>If the current thread is the holder of this lock then
         * the hold count is decremented. If the hold count is now
         * zero then the lock is released.  If the current thread is
         * not the holder of this lock then {@link
         * IllegalMonitorStateException} is thrown.
         *
         * @throws IllegalMonitorStateException if the current thread does not
         * hold this lock
         */
        public void unlock() {
            sync.release(1);
        }


#### 写锁尝试加锁
WriteLock#tryLock

```java
   public boolean tryLock( ) {
            return sync.tryWriteLock();
   }
```



```java
        /**
         * Performs tryLock for write, enabling barging in both modes.
         * This is identical in effect to tryAcquire except for lack
         * of calls to writerShouldBlock.
         */
        final boolean tryWriteLock() {
            Thread current = Thread.currentThread();
            int c = getState();
            if (c != 0) {
                int w = exclusiveCount(c);
                // 有读锁或者是当前写锁线程不是自身, 则改状态失败, 去阻塞
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;
                if (w == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
            }
            if (!compareAndSetState(c, c + 1))
                return false;
            setExclusiveOwnerThread(current);
            return true;
        }

```

## 总结

每个ReetrantReadWriteLock 
1. 包含三个内部类ReadLock,WriteLock,Sync(NofairSync/FairSync),
    前两个会依赖后面的Sync实现, 并起到封装api的作用;
2. 读写锁的实现都依赖同一个sync中的变量; 
    1. 读写锁不同的线程访问并修改同一个变量， 可能带来缓存行失效的问题;

