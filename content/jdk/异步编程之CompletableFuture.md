---
title: "异步编程之CompletableFuture"
date: 2020-12-30T21:39:16+08:00
draft: false
tags: ["并发"]

---

Future 接口介绍

JDK5 新增了 Future 接口，用于描述一个异步计算的结果。虽然 Future 以及相关使用方法提供了异步执行任务的能力，但是对于结果的获取却是很不方便，只能通过阻塞或者轮询的方式得到任务的结果。

1. 阻塞的方式显然和我们的异步编程的初衷相违背，
2. 轮询的方式又会耗费无谓的 CPU 资源，而且也不能及时地得到计算结果，

为什么不能用观察者设计模式呢？即当计算结果完成及时通知监听者。

有一些开源框架实现了我们的设想，例如 Netty 的 ChannelFuture 类扩展了 Future 接口，通过提供 addListener 方法实现支持回调方式的异步编程。Netty 中所有的 I/O 操作都是异步的,这意味着任何的 I/O 调用都将立即返回，而不保证这些被请求的 I/O 操作在调用结束的时候已经完成。取而代之地，你会得到一个返回的 ChannelFuture 实例，这个实例将给你一些关于 I/O 操作结果或者状态的信息。当一个 I/O 操作开始的时候，一个新的 Future 对象就会被创建。在开始的时候，新的 Future 是未完成的状态－－它既非成功、失败，也非被取消，因为 I/O 操作还没有结束。如果 I/O 操作以成功、失败或者被取消中的任何一种状态结束了，那么这个 Future 将会被标记为已完成，并包含更多详细的信息（例如：失败的原因）。请注意，即使是失败和被取消的状态，也是属于已完成的状态。

下面主要介绍下CompletableFuture的实现和相关源码解析;

### 同步执行动作

#### 示例

```java
static void thenApplyExample() {
    CompletableFuture<String> cf = CompletableFuture.completedFuture("message").thenApply(s -> {
    assertFalse(Thread.currentThread().isDaemon());
    returns.toUpperCase();
    });
    assertEquals("MESSAGE", cf.getNow(null));
}
```

以上代码在计算正常完成的前提下将执行动作（此处为转换成大写字母）。

#### 相关源码

1. 接收任务API

   ```java
       /**
        * 对于当前的{@link CompletableFuture} 应用另外一个函数
        * @param fn
        * @param <U>
        * @return
        */
       public <U> CompletableFuture<U> thenApply(
           Function<? super T,? extends U> fn) {
           // 不定义线程池的情况下去执行 fn;
           return uniApplyStage(null, fn);
       }
   ```

   如上uniApplyStage第一个线程池参数位null, 所以是同步运行模式;
2. 运行或者是封装Completion到队列

   ```java
       private <V> CompletableFuture<V> uniApplyStage(
           Executor e, Function<? super T,? extends V> f) {
           // 1. 检测依赖函数不能为null 
           if (f == null) throw new NullPointerException();
           // 2. 在当前Future中, 触发执行，并输出结果到 d
           CompletableFuture<V> d =  new CompletableFuture<V>();
           // 执行目标过程 f
           // 两种情况封装任务UniApply执行 
           // 2.1. 定义了线程池e      2.2 再d中同步运行失败;
           if (e != null || !d.uniApply(this, f, null)) {
               // 定义一个任务, 去线程池中e中, 针对线程池中this 中执行f 并输出到d中;
               UniApply<T,V> c = new UniApply<T,V>(e, d, this, f);
               // 将任务c押入栈中;
               push(c);
               //  触发任务完成;
               c.tryFire(SYNC);
           }
           /**
            *   返回融合后的{@link CompletableFuture}
            */
           return d;
       }

   ```

   根据例子中的代码

   1. 不走线程池的情况下, 在目标Future中执行, 并得到结果
3. 直接在新 CompletableFuture -- d 中运行

   ```java
       /**
        * 从依赖的{@link CompletableFuture} 中获取结果, 再执行函数f, 并拿到结果给到当前的{@link CompletableFuture}
        *
        * @param a  数据来源 {@link CompletableFuture}
        * @param f  即将引用的函数
        * @param c
        * @param <S>
        * @return
        */
       final <S> boolean uniApply(CompletableFuture<S> a,
                                  Function<? super S,? extends T> f,
                                  UniApply<S,T> c) {
           Object r; Throwable x;
           // 1. 来源Future a如果未完成, 当前任务不会被执行, 所以这里直接返回false;
           if (a == null || (r = a.result) == null || f == null)
               return false;
           // 2. 来源Future 执行完成, 当前任务还没有拿到结果的情况下: 
           tryComplete: if (result == null) {

               // 2.1. 设置异常运行的结果;
               if (r instanceof AltResult) {
                   // 传播异常;
                   if ((x = ((AltResult)r).ex) != null) {
                       completeThrowable(x, r);
                       break tryComplete;
                   }
                   r = null;
               }

               try {
               // 2.2 如果有定义线程池, 则通过c将当前任务提交到线程池在重新执行;
                   // 定义了任务封装UniApply 并且如果里面有线程池, 
                   // 则会优先使用之, 去运行该任务, 此后直接返回;
                   if (c != null && !c.claim())
                       return false;
               // 2.3 没有定义线程池的情况下, 这里会同步执行任务 并设置任务的执行结果;                    
                   // 任务封装UniApply不能处理完成该任务, 则在调用线程执行该任务, 并完成之;
                   @SuppressWarnings("unchecked") S s = (S) r;
                   completeValue(f.apply(s));
               } catch (Throwable ex) {
               // 2.4 任务执行失败, 这里会设置任务异常;
                   // 设置异常运行的结果;
                   completeThrowable(ex);
               }
           }
           return true;
       }
   ```
4. 如果通过UniApply运行;
   参考下面的异步执行部分;

最后 thenApply 函数就能得到目标Future;

### 异步执行

#### 示例

```java
 static void thenApplyAsyncExample() {
    CompletableFuture<String>cf = CompletableFuture.completedFuture("message").thenApplyAsync(s -> {
    assertTrue(Thread.currentThread().isDaemon());
    randomSleep();
    returns.toUpperCase();
    });
    assertNull(cf.getNow(null));
    assertEquals("MESSAGE", cf.join());
}
```

#### 相关源码

1. 异步执行新的函数

   ```java
       public <U> CompletableFuture<U> thenApplyAsync(
           Function<? super T,? extends U> fn) {
           return uniApplyStage(asyncPool, fn);
       }
   ```
2. 默认依赖的线程池定义如下

   ```java
       /**
        * Default executor -- ForkJoinPool.commonPool() unless it cannot
        * support parallelism.
        */
       private static final Executor asyncPool = useCommonPool ?
           ForkJoinPool.commonPool() : new ThreadPerTaskExecutor();

   ```
3. 回在到同步异步兼容调度函数下

   ```java
       private <V> CompletableFuture<V> uniApplyStage(
           Executor e, Function<? super T,? extends V> f) {

           if (f == null) throw new NullPointerException();
           // 目标
           CompletableFuture<V> d =  new CompletableFuture<V>();
           // 2. 执行目标过程 f
           // 两种情况 2.1. 定义了线程池e      2.2 再d中同步运行失败;
           if (e != null || !d.uniApply(this, f, null)) {
               // 定义一个任务, 去线程池中e中, 针对线程池中this 中执行f 并输出到d中;
               UniApply<T,V> c = new UniApply<T,V>(e, d, this, f);
               // 将任务c押入栈中;
               push(c);
               //  触发任务完成;
               c.tryFire(SYNC); // 同步模式该方法返回null
           }
           /**
            *   返回融合后的{@link CompletableFuture}
            */
           return d;
       }
   ```

这里显然走2.1分支
将当前任务封装成UniApply<T,V> 押入栈中后 并触发之;

4. UniApply 还是回到把任务的执行转发到目标 CompletableFuture (参数有函数和参数两部分)

```java

    @SuppressWarnings("serial")
    static final class UniApply<T,V> extends UniCompletion<T,V> {
        Function<? super T,? extends V> fn;
        UniApply(Executor executor, CompletableFuture<V> dep,
                 CompletableFuture<T> src,
                 Function<? super T,? extends V> fn) {
            super(executor, dep, src); this.fn = fn;
        }

        /**
         *
         * @param mode 指定触发模式;
         * @return
         */
        final CompletableFuture<V> tryFire(int mode) {
            // 依赖的完成器(目标)   ;   数据源来源
            CompletableFuture<V> d; CompletableFuture<T> a;
            // 1. 运行, 运行失败直接返回null;
            if ((d = dep) == null ||
                //2.  转发任务的执行到目标CompletableFuture: 只有异步才是null (该任务自己自身提交到线程池时这里为ASYNC == 1)
                !d.uniApply(a = src, fn, mode > 0 ? null : this)) // 主要是执行 f.apply(a)
                return null;
            // 3. 运行成功 重置任务;
            dep = null; src = null; fn = null;
            // 4. 再目标完成器中触发完成信号;
            return d.postFire(a, mode);  
        }
    }
```

5. 如下可知 目标Future中运行时又通过claim函数转发给了, 任务定义UniApply

```java
    /**
     * 从依赖的{@link CompletableFuture} 中获取结果, 再执行函数f, 并拿到结果给到当前的{@link CompletableFuture}
     *
     * @param a  数据来源 {@link CompletableFuture}
     * @param f  即将引用的函数
     * @param c
     * @param <S>
     * @return
     */
    final <S> boolean uniApply(CompletableFuture<S> a,
                               Function<? super S,? extends T> f,
                               UniApply<S,T> c) {
        Object r; Throwable x;
        if (a == null || (r = a.result) == null || f == null)
            return false;

        tryComplete: if (result == null) {

            // 1. 设置异常运行的结果;
            if (r instanceof AltResult) {
                // 传播异常;
                if ((x = ((AltResult)r).ex) != null) {
                    completeThrowable(x, r);
                    break tryComplete;
                }
                r = null;
            }

            try {
                // 2. 没有定义线程池, 或者其判定已经或者被其他线程运行了;
                if (c != null && !c.claim()) // 在线程池中调用c.tryFire(mode)
                    return false;
                // 3. 运行并设置结果;
                @SuppressWarnings("unchecked") S s = (S) r;
                completeValue(f.apply(s));
            } catch (Throwable ex) {
                // 4. 设置异常运行的结果;
                completeThrowable(ex);
            }
        }
        return true;
    }
```

6. claim 函数在UniApply的基类里面：

   1. 其保证该任务仅能被执行一次;
   2. 如果当前任务有设定线程池，则claim函数会将自己提交到自己的线程池里面，
      1. 任务会让依赖的CompletableFuture#uniApply执行, 不过此时第三个参数 UniApply<S,T> 为null,所以其会执行运行apply函数;

   ```java
       abstract static class UniCompletion<T,V> extends Completion {
           Executor executor;                 // executor to use (null if none) 执行器;
           CompletableFuture<V> dep;          // the dependent to complete  依赖的完成器;
           CompletableFuture<T> src;          // source for action  数据源;

           UniCompletion(Executor executor, CompletableFuture<V> dep,
                         CompletableFuture<T> src) {
               this.executor = executor; this.dep = dep; this.src = src;
           }

           /**
            * Returns true if action can be run. Call only when known to
            * be triggerable. Uses FJ tag bit to ensure that only one
            * thread claims ownership.  If async, starts as task -- a
            * later call to tryFire will run action.
            */
           final boolean claim() {
               Executor e = executor;
               if (compareAndSetForkJoinTaskTag((short)0, (short)1)) {
                   // 没有线程池, 则只修改了运行标示, 并不返回true;
                   if (e == null)
                       return true;
                   executor = null; // disable
                   // 在线程池中运行当前任务;
                   e.execute(this);
               }
               // 当前action不能再被运行;
               // 1. 修改标识位失败: 其他线程去运行了;
               // 2. 修改标识位成功: 但是存在线程池, 因此去线程池中运行了.
               return false;
           }

           final boolean isLive() { return dep != null; }
       }

   ```
7. 任务定义更是在基类里面

   ```java
       // Modes for Completion.tryFire. Signedness matters.
       static final int SYNC   =  0;
       static final int ASYNC  =  1;
       static final int NESTED = -1;

       @SuppressWarnings("serial")
       abstract static class Completion extends ForkJoinTask<Void>
           implements Runnable, AsynchronousCompletionTask {
           volatile Completion next;      // Treiber stack link

           /**
            * Performs completion action if triggered, returning a
            * dependent that may need propagation, if one exists.
            *
            * @param mode SYNC, ASYNC, or NESTED
            */
           abstract CompletableFuture<?> tryFire(int mode);

           /** Returns true if possibly still triggerable. Used by cleanStack. */
           abstract boolean isLive();

           public final void run()                { tryFire(ASYNC); }
           public final boolean exec()            { tryFire(ASYNC); return true; }
           public final Void getRawResult()       { return null; }
           public final void setRawResult(Void v) {}
       }

   ```

   如上UniApply最后又通过tryFire(ASYNC) 再一次将运行代码路由到了UniApply
   只是这次是同步模式;
   接下来就是如同步章节的处理过程;

   注意这两次tryFire调用

   1. 同步/异步模式: 会返回null
   2. NEST模式 在目标CompletableFuture中返回, 其会返回自身;
8. 由5.3 , 4.4可知任务完成后, 将通知相关等待方;

   ```JAVA
       /**
        * Post-processing by dependent after successful UniCompletion
        * tryFire.  Tries to clean stack of source a, and then either runs
        * postComplete or returns this to caller, depending on mode.
        */
       final CompletableFuture<T> postFire(CompletableFuture<?> a, int mode) {

           // 数据来源a 能够拿到结果, 
           if (a != null && a.stack != null) {
               // 
               if (mode < 0 || a.result == null)
                   a.cleanStack();
               else
                   a.postComplete();
           }
           // 当前数据源得到结果, 则可以提交该任务; 
           if (result != null && stack != null) {
               // 级联模式返回自己;
               if (mode < 0)
                   return this;
               // 触发自己的监听任务;  
               else
                   postComplete();
           }
           return null;
       }
   ```

### 使用固定的线程池完成异步执行动作示例

### 全部完成则响应

#### 入口代码:

```JAVA
   public static CompletableFuture<Void> allOf(CompletableFuture<?>... cfs) {
        return andTree(cfs, 0, cfs.length - 1);
    }
```

#### 递归计算

如下使用递归的方式构建节点树:

andTree 会将所有的 Future 构建一颗二叉树, 最终返回根节点.

1. 每个节点的完成都会依赖或者由其孩子节点触发;
   1. 每个孩子通过自己监听队列中的 Completion#tryFire(int mode) 触发父亲的 biRelay 方法
   2. 但是当前节点的 biRelay 方法, 依赖每个节点的完成，才完成，并触发postFire方法,  将完成信号级联传播到父亲节点;

递归重复子过程:

1. 构建依赖(输出) Future: d
2. 使用二分法切分左右子树到节点 a、b
3. 封装依赖d(当前节点)和被依赖a、b到 BiRelay, 并将后者压入a、b的栈中, 后期即可接受它们的回调, 能够达到回溯的效果;

```JAVA

    static CompletableFuture<Void> andTree(CompletableFuture<?>[] cfs,
                                           int lo, int hi) {
        // 当前需要构建的节点 d; 它将依赖子任务的运行完成, 并能够响应子任务的完成信号;                                                                               
        CompletableFuture<Void> d = new CompletableFuture<Void>();
        // 1. 递归边界条件, 当前节点直接完成; 
        if (lo > hi) // empty
            d.result = NIL;
        // 2. 构建当前节点d 依赖, 其孩子节点的完成;  
        else {
            CompletableFuture<?> a, b;
            // 2.1 中点
            int mid = (lo + hi) >>> 1;
            // 2.2 递归封装左子树
            if ((a = (lo == mid ? cfs[lo] :
                      andTree(cfs, lo, mid))) == null || 
                // 递归封装右子树                  
                (b = (lo == hi ? a : (hi == mid+1) ? cfs[hi] :
                      andTree(cfs, mid+1, hi)))  == null)
                throw new NullPointerException();

            //  2.3 组合左右子树, 让当前节点检测依赖的两个孩子, 如果两个孩子没有完成
            if (!d.biRelay(a, b)) {
                // 2.3.1 创建一个 BiCompletion, 它会封装d并代表 d 加入 a和b 的等待队列;
                BiRelay<?,?> c = new BiRelay<>(d, a, b);
                // 2.3.2 将 c 加入a和b的等待队列;
                a.bipush(b, c);
                // 2.3.3 BiCompletion c 让 组合Future d 去尝试检测其依赖Future的完成结果, 如果完成则通知自己的等待列表;
                c.tryFire(SYNC);
            }
        }
        // 3. 触发 Future 的完成;
        return d;
    }
```

### 任意完成则完成

#### 入口代码

根据输入的 future 构建递归树, 根节点就是输出的CompletableFuture;

```java
    public static CompletableFuture<Object> anyOf(CompletableFuture<?>... cfs) {
        return orTree(cfs, 0, cfs.length - 1);
    }
```

1. 封装依赖节点d, 被依赖节点a，b为任务 OrRelay
2. 将任务输出到依赖节点a，b 相当于加入它们的订阅列表;
3. 依赖任务执行完毕后会通知/回溯到依赖节点;

```java
    /** Recursively constructs a tree of completions. */
    static CompletableFuture<Object> orTree(CompletableFuture<?>[] cfs,
                                            int lo, int hi) {
        CompletableFuture<Object> d = new CompletableFuture<Object>();
        if (lo <= hi) {
            CompletableFuture<?> a, b;
            int mid = (lo + hi) >>> 1;
            if ((a = (lo == mid ? cfs[lo] :
                      orTree(cfs, lo, mid))) == null ||
                (b = (lo == hi ? a : (hi == mid+1) ? cfs[hi] :
                      orTree(cfs, mid+1, hi)))  == null)
                throw new NullPointerException();
            if (!d.orRelay(a, b)) {
                OrRelay<?,?> c = new OrRelay<>(d, a, b);
                a.orpush(b, c);
                c.tryFire(SYNC);
            }
        }
        return d;
    }
```

## 参考

1. [鸟窝](https://colobu.com/2016/02/29/Java-CompletableFuture/)
2. [IBM-通过实例理解 JDK8 的 CompletableFuture](https://developer.ibm.com/zh/articles/j-cf-of-jdk8/)
3. [youngitman](http://youngitman.tech/2019/02/13/completablefuture%E6%BA%90%E7%A0%81/)
4. [oracle](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletionStage.html)
