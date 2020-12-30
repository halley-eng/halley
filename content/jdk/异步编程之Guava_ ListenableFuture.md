title: "异步编程之CompletableFuture"
date: 2020-12-30T21:39:16+08:00
draft: true
tags: ["并发"]


## 创建 

### 通过包装线程池

1. 相对于 JDK 提供的 ExecutorService.submit(Callable) 形式去创建异步计算任务， 
2. Guava 提供 ListeningExecutorService  相对于将原来返回的 Future 切换成 ListenableFuture
3. 两者可以通过 MoreExecutors.listeningDecorator(ExecutorService) 实现转换


```JAVA

ListeningExecutorService service = MoreExecutors.listeningDecorator(Executors.newFixedThreadPool(10));
ListenableFuture<Explosion> explosion = service.submit(
    new Callable<Explosion>() {
      public Explosion call() {
        return pushBigRedButton();
      }
    });
Futures.addCallback(
    explosion,
    new FutureCallback<Explosion>() {
      // we want this handler to run immediately after we push the big red button!
      public void onSuccess(Explosion explosion) {
        walkAwayFrom(explosion);
      }
      public void onFailure(Throwable thrown) {
        battleArchNemesis(); // escaped the explosion!
      }
    },
    service);

```

### 直接包装 FutureTask

1. ListenableFutureTask.create(Callable<V>) 
2. ListenableFutureTask.create(Runnable, V)

注意 ListenableFutureTask 不支持直接继承

如果你期望通过抽象直接设置Future的值，而不是通过某个方法计算出值，可以考虑继承 AbstractFuture<V> 或者直接使用 SettableFuture

如果需要将 Future 转换为 一个 ListenableFuture , 只能选择 使用  JdkFutureAdapters.listenInPoolThread(Future)


## 应用 

### 链式异步调用 transformAsync 


```JAVA
ListenableFuture<RowKey> rowKeyFuture = indexService.lookUp(query);
AsyncFunction<RowKey, QueryResult> queryFunction =
  new AsyncFunction<RowKey, QueryResult>() {
    public ListenableFuture<QueryResult> apply(RowKey rowKey) {
      return dataService.read(rowKey);
    }
  };
ListenableFuture<QueryResult> queryFuture =
    Futures.transformAsync(rowKeyFuture, queryFunction, queryExecutor);

```

### 链式同步调用 transform(ListenableFuture<A>, Function<A, B>, Executor)

### 封装一组 Future, allAsList(Iterable<ListenableFuture<V>>) 快速失败

### 封装一组 Future successfulAsList(Iterable<ListenableFuture<V>>) 失败取消为null


### 避免Future 嵌套


```JAVA
executorService.submit(new Callable<ListenableFuture<Foo>() {
  @Override
  public ListenableFuture<Foo> call() {
    return otherExecutorService.submit(otherCallable);
  }
});

```

如果外层Future的取消, 并不能传递到内层Future , 所以Future不能简单包装;











### 参考

https://github.com/google/guava/wiki/ListenableFutureExplained