---
title: "并发编程框架 CountDown"
date: 2020-11-25T21:38:02+08:00
draft: true
---



```java
    private static final class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 4982264981922014374L;

        Sync(int count) {
            setState(count);
        }

        int getCount() {
            return getState();
        }

        protected int tryAcquireShared(int acquires) {
            // 知道当前剩余可获取信号量为0的时候 才能获取到共享锁;
            return (getState() == 0) ? 1 : -1;
        }

        protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
                // 拿状态
                int c = getState();
                // 如果为0 则已经释放过了, 这次直接释放失败, 保证只释放一次
                if (c == 0)
                    return false;
                // 设置下一次释放后锁的数量;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
    }
```


用户调用countdown函数, 底层是释放共享锁

```java
    public void countDown() {
        sync.releaseShared(1);
    }

```

此时会间接调用

```java
        protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
                // 拿状态
                int c = getState();
                // 如果为 0 则已经释放过了, 这次直接释放失败, 保证只释放一次
                if (c == 0)
                    return false;
                // 设置下一次释放后锁的数量;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
```


修改后的状态如果是0，则会唤醒等待线程去获取共享锁
```java
     protected int tryAcquireShared(int acquires) {
            // 直到当前剩余可获取信号量为0的时候 才能获取到共享锁;
            return (getState() == 0) ? 1 : -1;
        }
```

那么上面肯定也可以获取到

则await当前CountDown的线程会被唤醒;


