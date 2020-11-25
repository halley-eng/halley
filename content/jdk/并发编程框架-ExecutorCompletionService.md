---
title: "并发编程框架 ExecutorCompletionService"
date: 2020-11-25T21:44:15+08:00
draft: false
---


#### 提交任务

```JAVA
    public Future<V> submit(Callable<V> task) {
        if (task == null) throw new NullPointerException();
        
        // 创建任务;
        RunnableFuture<V> f = newTaskFor(task);
        // 封装任务并将其提交到执行器;
        executor.execute(new QueueingFuture(f));
        return f;
    }
```

封装任务是为了在任务执行完毕后能够接收任务完成信号

```JAVA
    /**
     * FutureTask extension to enqueue upon completion
     */
    private class QueueingFuture extends FutureTask<Void> {
        QueueingFuture(RunnableFuture<V> task) {
            super(task, null);
            this.task = task;
        }
        // 任务完成后 将任务添加到完成队列; 
        protected void done() { completionQueue.add(task); }
        private final Future<V> task;
    }
```




### 完成任务的获取

任务的获取主要是封装了queue的api 支持三种模式

```JAVA
    
    // 获取不到将阻塞获取
    public Future<V> take() throws InterruptedException {
        return completionQueue.take();
    }

    // 查询不到返回null
    public Future<V> poll() {
        return completionQueue.poll();
    }
    
    // 待超时的获取;
    public Future<V> poll(long timeout, TimeUnit unit)
            throws InterruptedException {
        return completionQueue.poll(timeout, unit);
    }
    
```


#### 总结

ExecutorCompletionService  通过封装原始的任务到 QueueingFuture

1. QueueingFuture是一种FutureTask实现, 另外覆写了done方法, 
   可以做到当任务完成时, 将被封装的任务压入队列,并等待消费; 



