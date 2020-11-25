---
title: "并发编程框架 CyclicBarrier"
date: 2020-11-25T21:37:30+08:00
draft: false
---

![ss](https://cdn.jsdelivr.net/gh/halley-eng/halley/static/images/jdk/concurrent/cyclicbarrier.jpg)

### 数据结构



```java
    /**
     * 锁屏障
     * The lock for guarding barrier entry */
    private final ReentrantLock lock = new ReentrantLock();
    /**
     * 条件变量
     * Condition to wait on until tripped */
    private final Condition trip = lock.newCondition();

    /** 参与方数量 The number of parties */
    private final int parties;
    /*  各参与方都到位后的触发命令 The command to run when tripped */
    private final Runnable barrierCommand;
    /** 当前代 The current generation */
    private Generation generation = new Generation();

    /**
     * 当前代未到位的参与方数量
     * Number of parties still waiting. Counts down from parties to 0
     * on each generation.  It is reset to parties on each new
     * generation or when broken.
     */
    private int count;
```


### await

各参与方通过调用await相关方法
1. 使用同步锁lock
2. 依次到达屏障处, 除非最后一个到达都会阻塞在屏障处; 
3. 最后一个到达后执行后置命令, 并退出;


```java
    /**
     * Main barrier code, covering the various policies.
     */
    private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
        // 1. 各个线程以同步的方式进入到循环屏障;
        final ReentrantLock lock = this.lock;
        lock.lock();

        try {
            final Generation g = generation;
            // 2. 第一次检测当前代的broken标示;
            if (g.broken)
                throw new BrokenBarrierException();

            // 3. 响应中断
            if (Thread.interrupted()) {
                breakBarrier();
                throw new InterruptedException();
            }

            // 4. 当前是最后一个线程, 去执行相应的action, 最后一个线程不用去加入条件队列进入阻塞的流程;
            int index = --count;  // 更新剩余等待的线程数目；
            if (index == 0) {  // tripped
                boolean ranAction = false;
                try {
                    //
                    final Runnable command = barrierCommand;
                    if (command != null)
                        command.run();

                    ranAction = true; // 标记已经运行过action
                    nextGeneration(); // 唤醒其他人, 并重新初始化下一代;
                    return 0;
                } finally {
                    if (!ranAction)
                        breakBarrier();
                }
            }

            // loop until tripped, broken, interrupted, or timed out
            // 5. 循环直到broken，中断或超时
            for (;;) {
                try {
                    // 5.1 没有设置超时时间, 直接丢进条件队列, 需要等待该条件被notify
                    if (!timed)
                        trip.await();
                    // 5.2 设置超时时间的等待: 支持超时响应的加入条件队列;
                    else if (nanos > 0L) // 继续等待;
                        nanos = trip.awaitNanos(nanos);

                } catch (InterruptedException ie) {
                    // 5.3 中断退出;
                    if (g == generation && ! g.broken) {
                        breakBarrier(); // 更新当前代标示, 并触发各等待线程继续执行;
                        throw ie;   // 抛出异常;
                    } else {
                    // 5.4
                        // We're about to finish waiting even if we had not
                        // been interrupted, so this interrupt is deemed to
                        // "belong" to subsequent execution.
                        Thread.currentThread().interrupt();
                    }
                }

                // 5.5 第二次检测broken标示,
                if (g.broken)
                    throw new BrokenBarrierException();
                // 5.6 当前处理代被异常结束；
                if (g != generation)
                    return index;

                // 5.7 设置了超时时间, 并且超时则标记当前代结束，并开启下一代;
                if (timed && nanos <= 0L) {
                    breakBarrier();
                    throw new TimeoutException();
                }
            }

        } finally {
            // 解锁;
            lock.unlock();
        }
    }

```



### 重置锁屏障

1. 获取锁
1. breakBarrier: 对当前代的Generate添加中断表示;
2. nextGeneration: 开启下一代，并唤醒


```java
    
     /**
     * Resets the barrier to its initial state.  If any parties are
     * currently waiting at the barrier, they will return with a
     * {@link BrokenBarrierException}. Note that resets <em>after</em>
     * a breakage has occurred for other reasons can be complicated to
     * carry out; threads need to re-synchronize in some other way,
     * and choose one to perform the reset.  It may be preferable to
     * instead create a new barrier for subsequent use.
     */
    public void reset() {
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

1. 更新broken标示
1. 唤醒当前代所有的等待者
    1. 这些等待着会被唤醒, 然后检测broken标示, 并因此抛出BrokenBarrierException异常

```java
    /**
     * 将当前的障碍生成设置为已破坏并唤醒相关方。
     * 仅在保持锁定状态下调用。
     * Sets current barrier generation as broken and wakes up everyone.
     * Called only while holding lock.
     */
    private void breakBarrier() {
        generation.broken = true;
        count = parties;
        trip.signalAll();
    }

```


开始下一代, 其中会
1. 先唤醒上一代所有的等待者
    1. 这些等待着会被唤醒, 然后检测broken标示, 并因此抛出BrokenBarrierException异常
2. 重新初始化下一代上下文;

```java
    /**
     * Updates state on barrier trip and wakes up everyone.
     * Called only while holding lock.
     */
    private void nextGeneration() {
        // signal completion of last generation
        trip.signalAll();
        // set up next generation
        count = parties;
        generation = new Generation();
    }
```

### Demo



```java
  
 class Solver {
   final int N;
   final float[][] data;
   final CyclicBarrier barrier;
    
   // 每个工作对象处理一行, 之后进入await, 
   class Worker implements Runnable {
     int myRow;
     Worker(int row) { myRow = row; }
     public void run() {
       while (!done()) {
         processRow(myRow);

         try {
           barrier.await();
         } catch (InterruptedException ex) {
           return;
         } catch (BrokenBarrierException ex) {
           return;
         }
       }
     }
   }

   public Solver(float[][] matrix) {
     data = matrix;
     N = matrix.length;
     Runnable barrierAction =
       new Runnable() { 
            public void run() { mergeRows(...); }
       };
     barrier = new CyclicBarrier(N, barrierAction);

     List<Thread> threads = new ArrayList<Thread>(N);
     for (int i = 0; i < N; i++) {
       Thread thread = new Thread(new Worker(i));
       threads.add(thread);
       thread.start();
   }
```

### 总结

CylicBarrier 对外方法只有await, reset 两个。 共同实现每count次触发一次屏障命令command, 并唤醒相关方继续执行的操作;

