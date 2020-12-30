---
title: "ThreadPollExecutor 源码分析"
date: 2020-12-30T21:39:16+08:00
draft: false
tags: ["并发"]
---

# ThreadPollExecutor 源码分析

![ThreadPoolExecutor](https://halley-image.oss-cn-beijing.aliyuncs.com/picgoThreadPoolExecutor.png)

## JDK版本约定
    1.8
## 提交任务

```JAVA

public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        int c = ctl.get();
        // 1. 不超过核心线程数, 创建新的worker运行;
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        // 2. 超过核心线程加入队列
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        // 3. 队列已满拒绝之;
        else if (!addWorker(command, false))
            reject(command);
    }

```
### AbstractExecutorService#submit(java.lang.Runnable, T) 封装 FutureTask

1. 任务被封装为 FutureTask
   1. 负责拦截异常和执行结果, 并提供await机制, 内部执行完毕后通知相关方;

```java

    protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
        return new FutureTask<T>(runnable, value);
    }
    protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        return new FutureTask<T>(callable);
    }
    // 提交被 FutureTask 封装后的任务
    public Future<?> submit(Runnable task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<Void> ftask = newTaskFor(task, null);
        execute(ftask);
        return ftask;
    }
    public <T> Future<T> submit(Runnable task, T result) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task, result);
        execute(ftask);
        return ftask;
    }
    public <T> Future<T> submit(Callable<T> task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task);
        execute(ftask);
        return ftask;
    }

```

### 创建worker

1. 乐观锁更新线程数
2. 创建worker, 关联firstTask, 并启动worker


```JAVA
    private boolean addWorker(Runnable firstTask, boolean core) {

        // 1. 保证线程数+1
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);
            // 1.1 线程池被关闭那么, 直接短路不去创建;
            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;
            // 1.2 乐观更新线程数;
            for (;;) {
                // 1.2.1 保证线程数不超
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                // 1.2.2 增加线程数;    
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                // 1.2.3 检测运行状态改变, 重新运行外层循环;     
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            // 2. 创建Worker, 如果创建失败需要回滚线程数
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                // 2.1 并发写安全;
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());
                    // 2.1 加入 worker 列表;
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        // 2.1.1 加入工作列表;    
                        workers.add(w);
                        // 2.1.2 修正最大线程数标记;
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        // 2.1.3 标记 worker 成功标记    
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                // 启动工作线程
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            // 3. 没有加入worker 或者是启动失败, 那么需要回撤;
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```  

### worker 实现分析

1. 作为ThreadPoolExecutor 的内部类, 共享其内部变量;
2. interruptIfStarted 中断工作线程
3. 继承AbstractQueuedSynchronizer, 简化任务执行过程中申请和释放锁资源, 
    保证同一个worker 只能由一个线程调度;
4. 上一个小节启动worker后, 调用Worker#run, 实际又转发给 ThreadPoolExecutor#runWorker, 
    后者会在循环中不断获取任务并执行之, 拦截异常 最终通过钩子函数 afterExecute 传递异常;

````JAVA
    private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
    {
        /**
         * This class will never be serialized, but we provide a
         * serialVersionUID to suppress a javac warning.
         */
        private static final long serialVersionUID = 6138294804551838833L;

        /** Thread this worker is running in.  Null if factory fails. */
        final Thread thread;
        /** Initial task to run.  Possibly null. */
        Runnable firstTask;
        /** Per-thread task counter */
        volatile long completedTasks;

        /**
         * Creates with given first task and thread from ThreadFactory.
         * @param firstTask the first task (null if none)
         */
        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            // 由线程工厂创建线程;
            this.thread = getThreadFactory().newThread(this);
        }

        /** Delegates main run loop to outer runWorker  */
        public void run() {
            runWorker(this);
        }

        // Lock methods
        //
        // The value 0 represents the unlocked state.
        // The value 1 represents the locked state.

        protected boolean isHeldExclusively() {
            return getState() != 0;
        }
        // 获取锁
        protected boolean tryAcquire(int unused) {
            if (compareAndSetState(0, 1)) {
                // 当前线程为独占线程
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }
        // 释放锁
        protected boolean tryRelease(int unused) {
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        public void lock()        { acquire(1); }
        public boolean tryLock()  { return tryAcquire(1); }
        public void unlock()      { release(1); }
        public boolean isLocked() { return isHeldExclusively(); }

        void interruptIfStarted() {
            Thread t;
            if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                }
            }
        }
    }
```


### 任务的运行 ThreadPoolExecutor#runWorker

1. 循环获取任务并执行之
    1.1  任务加锁
    1.2  执行
    1.3  捕获异常
    1.4  通过钩子函数 afterExecute 传播异常

```JAVA
    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            // 1. 循环子过程迭代获取任务;
            while (task != null || (task = getTask()) != null) {
                // 1.1 获取 worker 的锁, 保证该worker同时只有一个线程参与调度的过程;
                w.lock();
                // 1.2 
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    // 1.3 前置运行钩子函数;
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                    // 1.4 运行任务;    
                        task.run();
                    // 1.5 三张任务异常类型捕获和记录    
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                    // 1.6 钩子函数用来通知, 任务和异常结果;
                        afterExecute(task, thrown);
                    }
                } finally {
                    // 1.7 释放该 worker 的锁;
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            // 2. 任务执行完成后 后续统计工作;
            processWorkerExit(w, completedAbruptly);
        }
    }

```


## 钩子函数的使用

### afterExecute
```JAVA

 class ExtendedExecutor extends ThreadPoolExecutor {
   // ...
   protected void afterExecute(Runnable r, Throwable t) {
     super.afterExecute(r, t);
     
     if (t == null && r instanceof Future<?>) {
       try {
         // 检测结果  
         Object result = ((Future<?>) r).get();
         // 处理异常 
       } catch (CancellationException ce) {
           t = ce;
       } catch (ExecutionException ee) {
           t = ee.getCause();
       } catch (InterruptedException ie) {
           Thread.currentThread().interrupt(); // ignore/reset
       }
     }
     if (t != null)
       System.out.println(t);
   }
 }

 ```

## 总结

综上, 总结了 ThreadPoolExecutor 通过 execute 和 submit 函数提交任务的过程
设计到 Worker 的创建和执行; 

