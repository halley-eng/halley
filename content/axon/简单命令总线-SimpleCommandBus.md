---
title: "简单命令总线 SimpleCommandBus"
date: 2020-11-25T23:35:04+08:00
draft: false
---


### 架构图
![ss](https://cdn.jsdelivr.net/gh/halley-eng/halley/static/images/axon/1.jpg)

### 构建处理器 Builder

支持配置项如下:
1. 事务管理器: transactionManager
2. 监控器:  messageMonitor
3. 回滚配置: rollbackConfiguration
4. 冲突检测器: duplicateCommandHandlerResolver
5. 默认命令处理器: defaultCommandCallback 

它们的默认配置如下:

```JAVA
        private TransactionManager transactionManager = NoTransactionManager.INSTANCE;
        private MessageMonitor<? super CommandMessage<?>> messageMonitor = NoOpMessageMonitor.INSTANCE;
        private RollbackConfiguration rollbackConfiguration = RollbackConfigurationType.UNCHECKED_EXCEPTIONS;
        private DuplicateCommandHandlerResolver duplicateCommandHandlerResolver =
                DuplicateCommandHandlerResolution.logAndOverride();
        private CommandCallback<Object, Object> defaultCommandCallback = LoggingCallback.INSTANCE;

```

这里提供的默认处理器是什么都不做, 但是调用方就不用处理NPE问题了; 
比如事务处理器

```JAVA
public enum NoTransactionManager implements TransactionManager {

    /**
     * Singleton instance of the TransactionManager
     */
    INSTANCE;

    /**
     * Returns the singleton instance of this TransactionManager
     *
     * @return the singleton instance of this TransactionManager
     */
    public static TransactionManager instance() {
        return INSTANCE;
    }

    @Override
    public Transaction startTransaction() {
        return TRANSACTION;
    }

    private static final Transaction TRANSACTION = new Transaction() {
        @Override
        public void commit() {
            //no op
        }

        @Override
        public void rollback() {
            //no op
        }
    };
}
```



### 订阅 (subscribe)
        
对冲突处理提供通过自定义冲突处理器，提供定制化能力。 

```JAVA
    /**
     * 将处理器添加到订阅表, 并提供取消的功能;
     * Subscribe the given {@code handler} to commands with given {@code commandName}. If a subscription already
     * exists for the given name, the configured {@link DuplicateCommandHandlerResolver} will resolve the command
     * handler which should be subscribed.
     */
    @Override
    public Registration subscribe(String commandName, MessageHandler<? super CommandMessage<?>> handler) {
        assertNonNull(handler, "handler may not be null");
        subscriptions.compute(commandName, (k, existingHandler) -> {
            // 不存在或者是同一个
            if (existingHandler == null || existingHandler == handler) {
                return handler;
            } else {
            // 存在并且不是同一个, 则产生了冲突, 交给冲突处理器解决
                return duplicateCommandHandlerResolver.resolve(commandName, existingHandler, handler);
            }
        });
        // 输出取消方法
        return () -> subscriptions.remove(commandName, handler);
    }

```


### 处理消息



接收分发消息的指令

```JAVA
    @Override
    public <C> void dispatch(CommandMessage<C> command) {
        dispatch(command, defaultCommandCallback);
    }

    @Override
    public <C, R> void dispatch(CommandMessage<C> command, final CommandCallback<? super C, ? super R> callback) {
        doDispatch(intercept(command), callback);
    }
```

如上首先会经过分发拦截器

```JAVA
    @SuppressWarnings("unchecked")
    protected <C> CommandMessage<C> intercept(CommandMessage<C> command) {
        CommandMessage<C> commandToDispatch = command;
        // 迭代所有拦截器，依次处理该命令消息; 
        for (MessageDispatchInterceptor<? super CommandMessage<?>> interceptor : dispatchInterceptors) {
            commandToDispatch = (CommandMessage<C>) interceptor.handle(commandToDispatch);
        }
        return commandToDispatch;
    }
```

之后会去拿消息处理器并处理之

```JAVA
    protected <C, R> void doDispatch(CommandMessage<C> command, CommandCallback<? super C, ? super R> callback) {
        MessageMonitor.MonitorCallback monitorCallback = messageMonitor.onMessageIngested(command);

        // 查询消息处理器
        // 查到: 则处理
        Optional<MessageHandler<? super CommandMessage<?>>> optionalHandler = findCommandHandlerFor(command);
        if (optionalHandler.isPresent()) {
            handle(command, optionalHandler.get(), new MonitorAwareCallback<>(callback, monitorCallback));
        // 不能查询到: 异常之;
        } else {
            NoHandlerForCommandException exception = new NoHandlerForCommandException(
                    format("No handler was subscribed to command [%s]", command.getCommandName()));
            monitorCallback.reportFailure(exception);
            callback.onResult(command, asCommandResultMessage(exception));
        }
    }
```


实际处理方法如下
1. 第一层封装: 处理单元 UnitOfWork
    1. 消息
    2. 事务管理器
2. 第二层封装: 调用链 DefaultInterceptorChain
    1. 处理器拦截器
    2. 处理器

二层封装后通过处理单元发起调用链执行
```JAVA
    unitOfWork.executeWithResult(chain::proceed, rollbackConfiguration)
```

执行完毕后通过统一结果处理, 统一消息格式为 CommandResultMessage


详细源码如下： 
```JAVA
    /**
     * Performs the actual handling logic.
     *
     * @param command  The actual command to handle
     * @param handler  The handler that must be invoked for this command
     * @param callback The callback to notify of the result
     * @param <C>      The type of payload of the command
     * @param <R>      The type of result expected from the command handler
     */
    protected <C, R> void handle(CommandMessage<C> command,
                                 MessageHandler<? super CommandMessage<?>> handler,
                                 CommandCallback<? super C, ? super R> callback) {
        if (logger.isDebugEnabled()) {
            logger.debug("Handling command [{}]", command.getCommandName());
        }
        // 该命令被封装成一个工作单元
        UnitOfWork<CommandMessage<?>> unitOfWork = DefaultUnitOfWork.startAndGet(command);
        unitOfWork.attachTransaction(transactionManager);
        // chain 会负责依次调用 拦截器, 处理工作单元, 最后在调用handler
        InterceptorChain chain = new DefaultInterceptorChain<>(unitOfWork, handlerInterceptors, handler);

        // 最外层还是让unitOfWork发起调用;
        CommandResultMessage<R> resultMessage =
                asCommandResultMessage(unitOfWork.executeWithResult(chain::proceed, rollbackConfiguration));
        callback.onResult(command, resultMessage);
    }
```


### 总结

SimpleCommandBus 负责对命令消息进行派发处理，处理过程中的核心点如下
1. 封装事务管理器和消息等资源为处理单元UnitOfWork
    1. 事务管理器绑定到UnitOfWork实际是将其主要函数，通过函数封装，配置并参与到UnitOfWork的生命周期的过程; 
2. 封装处理器为Chain
3. 最外层的UnitOfWork作为执行模板，约定最外层的执行流程，执行过程会依次执行chain中的拦截器，最后执行handler; 


