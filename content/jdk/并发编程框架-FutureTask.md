---
title: "并发编程框架 FutureTask"
date: 2020-11-25T21:30:47+08:00
draft: true
---




### Future模式

Future代表异步执行结果. 用户可以
1. 调用get方法获取执行结果
2. 调用cancel方法取消执行
3. 调用get(long, TimeUnit) 支持超时的获取结果;

内存一致性保证:
   异步计算 happen-before 另外一个线程的Future.get
   
###  RunnableFuture<V> 实现

用于定义一种Future, 同时也是一个Runnable. 
一旦改Runnable运行完成, 那么该Future就能得到结果;


```java
/**
 * A {@link Future} that is {@link Runnable}. Successful execution of
 * the {@code run} method causes completion of the {@code Future}
 * and allows access to its results.
 * @see FutureTask
 * @see Executor
 * @since 1.6
 * @author Doug Lea
 * @param <V> The result type returned by this Future's {@code get} method
 */
public interface RunnableFuture<V> extends Runnable, Future<V> {
    /**
     * Sets this Future to the result of its computation
     * unless it has been cancelled.
     */
    void run();
}

```

### FutureTask 实现

在FutureTask的使用场景中:
1. 任务的执行和任务的执行状况查看在不同的线程
2. 不同的线程通过状态变量state, 保证并发安全, 达到协同工作的目的;
3. 任何修改状态的操作, 都要检测前置状态是否正确, 并使用CAS修改之;


#### 状态定义

```java
    /**
     * The run state of this task, initially NEW.  The run state
     * transitions to a terminal state only in methods set,
     * setException, and cancel.  During completion, state may take on
     * transient values of COMPLETING (while outcome is being set) or
     * INTERRUPTING (only while interrupting the runner to satisfy a
     * cancel(true)). Transitions from these intermediate to final
     * states use cheaper ordered/lazy writes because values are unique
     * and cannot be further modified.
     *
     * Possible state transitions:
     * NEW -> COMPLETING -> NORMAL
     * NEW -> COMPLETING -> EXCEPTIONAL
     * NEW -> CANCELLED
     * NEW -> INTERRUPTING -> INTERRUPTED
     */
    private volatile int state;
    private static final int NEW          = 0;
    private static final int COMPLETING   = 1;
    private static final int NORMAL       = 2;
    private static final int EXCEPTIONAL  = 3;
    private static final int CANCELLED    = 4;
    private static final int INTERRUPTING = 5;
    private static final int INTERRUPTED  = 6;
```
#### 任务的执行

```java
    public void run() {
        // 当前不是NEW或者给当前任务绑定当前线程执行失败;
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
        // 已经确保 执行依赖的状态当前为NEW, 并且给当前任务绑定当前线程执行中;
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    // 触发任务执行;
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                // 设置异常执行结果
                    result = null;
                    ran = false;
                    setException(ex);
                }
                // 设置正常训醒结果
                if (ran)
                    set(result);
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // 设置中断执行结果;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            if (s >= INTERRUPTING) //
                handlePossibleCancellationInterrupt(s);
        }
    }

```


#### 设置正常运行结果

1. 状态从NEW推到COMPLETING
2. 设置运行结果
3. 状态从COMPLETING推到NORMAL
4. 唤醒等待节点;

```java
    /**
     * Sets the result of this future to the given value unless
     * this future has already been set or has been cancelled.
     *
     * <p>This method is invoked internally by the {@link #run} method
     * upon successful completion of the computation.
     *
     * @param v the value
     */
    protected void set(V v) {
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = v;
            UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
            // 唤醒等待节点;
            finishCompletion();
        }
    }

```

唤醒方法实现如下：

```java
    /**
     * Removes and signals all waiting threads, invokes done(), and
     * nulls out callable.
     */
    private void finishCompletion() {
        // assert state > COMPLETING;
        for (WaitNode q; (q = waiters) != null;) {
            // 从当前线程中的字段中赋值为null, 保证只能当前线程去唤醒其他节点;
            if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
                // 迭代链表并依次唤醒等待节点;
                for (;;) {
                    Thread t = q.thread;
                    if (t != null) {
                        q.thread = null;
                        LockSupport.unpark(t);
                    }
                    WaitNode next = q.next;
                    if (next == null)
                        break;
                    q.next = null; // unlink to help gc
                    q = next;
                }
                break;
            }
        }

        done();

        callable = null;        // to reduce footprint
    }
```


#### 设置异常执行结果


```java
    protected void setException(Throwable t) {
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = t;
            UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // final state
            finishCompletion();
        }
    }
    
```

#### 查询执行结果

```java
    /**
     * @throws CancellationException {@inheritDoc}
     */
    public V get() throws InterruptedException, ExecutionException {
        int s = state;
        // 如果未完成则等待;
        if (s <= COMPLETING)
            s = awaitDone(false, 0L);
        // 拿到执行结果;
        return report(s);
    }
```

等待结果返回: 

```java
    /**
     * Awaits completion or aborts on interrupt or timeout.
     *
     * @param timed true if use timed waits
     * @param nanos time to wait, if timed
     * @return state upon completion
     */
    private int awaitDone(boolean timed, long nanos)
        throws InterruptedException {
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        WaitNode q = null;
        boolean queued = false;
        for (;;) {
            // 当前线程被中断, 则响应之;
            if (Thread.interrupted()) {
                removeWaiter(q);
                throw new InterruptedException();
            }

            int s = state;
            // 已经拿到结果, 则返回最新状态;
            if (s > COMPLETING) {
                if (q != null)
                    q.thread = null;
                return s;
            }
            else if (s == COMPLETING) // cannot time out yet
                Thread.yield();
            // 未创建则创建之;
            else if (q == null)
                q = new WaitNode();
            // 未加入队列则加入队列: 原子的将新节点加入到链表头部;
            else if (!queued)
                queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                      q.next = waiters, q);
            // 支持超时检测，则检测超时, 并超时阻塞;
            else if (timed) {
                nanos = deadline - System.nanoTime();
                if (nanos <= 0L) {
                    removeWaiter(q);
                    return state;
                }
                LockSupport.parkNanos(this, nanos);
            }
            // 不支持超时检测, 则无限期阻;
            else
                LockSupport.park(this);
        }
    }
```

状态到终态后上面后返回结果, 并执行下面的方法:

```java
    private V report(int s) throws ExecutionException {
        Object x = outcome;
        // 正常状态返回结果;
        if (s == NORMAL)
            return (V)x;
        // 非正常终态抛出异常;
        if (s >= CANCELLED)
            throw new CancellationException();
        throw new ExecutionException((Throwable)x);
    }

```

### 总结

1. 认识FutureTask的7种状态
2. 最终get方法的三种结果
    1. 正常返回
    2. CancellationException
    3. ExecutionException




