---
title: "并发编程框架 Semaphore"
date: 2020-11-25T21:36:55+08:00
draft: false
---

![ss](https://cdn.jsdelivr.net/gh/halley-eng/halley/static/images/jdk/concurrent/semaphore.jpg)

### 同步器Sync实现

1. 将AQS中的state变量, 映射到Sempaphore的permit变量;
2. 使用for(;;) + case的形式写获取和释放信号量的逻辑 

```java
    /**
     * Synchronization implementation for semaphore.  Uses AQS state
     * to represent permits. Subclassed into fair and nonfair
     * versions.
     */
    abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 1192457210091910933L;

        // 1. 将AQS中的state变量, 映射到Sempaphore的permit变量;
        // 许可数存放在state字段里面;
        Sync(int permits) {
            setState(permits);
        }
        // 在将state翻译成permits
        final int getPermits() {
            return getState();
        }
        
        // 2. 使用for(;;) + case的形式写获取和释放信号量的逻辑 
        /* 非公平获取锁 */
        final int nonfairTryAcquireShared(int acquires) {
            for (;;) {
                // 确实没有剩下多少
                int available = getState();
                int remaining = available - acquires;
                // 尝试修改剩余许可证数量
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }

        protected final boolean tryReleaseShared(int releases) {
            for (;;) {
                int current = getState();
                int next = current + releases;
                if (next < current) // overflow
                    throw new Error("Maximum permit count exceeded");
                if (compareAndSetState(current, next))
                    return true;
            }
        }

        final void reducePermits(int reductions) {
            for (;;) {
                int current = getState();
                int next = current - reductions;
                if (next > current) // underflow
                    throw new Error("Permit count underflow");
                if (compareAndSetState(current, next))
                    return;
            }
        }

        final int drainPermits() {
            for (;;) {
                int current = getState();
                if (current == 0 || compareAndSetState(current, 0))
                    return current;
            }
        }
    }
```

#### 非公平


```java

    /**
     * NonFair version
     */
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = -2694183684443567898L;

        /**
         *  利用Sync中的同步变量 初始化permits数
         */
        NonfairSync(int permits) {
            super(permits);
        }

        /**
         * 尝试获取许可证
         */
        protected int tryAcquireShared(int acquires) {
            return nonfairTryAcquireShared(acquires);
        }
    }

```

### 公平同步器

公平同步器, 通过状态的修改必然依赖于队列中是否有已等待的节点;
来保证新线程, 不会和队列中的线程进行抢占来实现;

```java
    /**
     * Fair version
     */
    static final class FairSync extends Sync {
        private static final long serialVersionUID = 2014338818796000944L;

        FairSync(int permits) {
            super(permits);
        }

        protected int tryAcquireShared(int acquires) {
            for (;;) {
                if (hasQueuedPredecessors())
                    return -1;
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
    }
```


### 获取信号量

```java
   public void acquire(int permits) throws InterruptedException {
        if (permits < 0) throw new IllegalArgumentException();
        sync.acquireSharedInterruptibly(permits);
    }

```

### 释放信号量

```java
    public void release(int permits) {
        if (permits < 0) throw new IllegalArgumentException();
        sync.releaseShared(permits);
    }
```


### 总结

Semaphore是AQS框架的经典实现
1. 将AQS框架中的state翻译成premit
2. 依赖AQS共享锁API实现Semaphore; 
2. 提供非公平和公平两种同步器
3. AQS会依赖以上同步器的 tryAcquireShared，tryReleaseShared 方法,实现超时检测和中断响应版本的acquire或者是release api;






