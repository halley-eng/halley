---
title: "Axon总揽"
date: 2020-11-25T23:34:02+08:00
draft: false
---



### Messaging Concepts(消息概念)


Axon 核心 概念之一是使用消息来做组件中的通信, 这样可以使得组件间位置透明, 以此做到灵活扩缩容; 

所有的消息都实现Message接口, 不同的类型的消息可以做到明显区分

1. 所有消息包含 payload、meta data 和 unique identifier 三个部分;
2. payload: 消息体
    1. 主要是包含类型 和 对象数据组成; 
3. meta data: 描述消息体的上下文
   1. 可以用来存储tracing信息 -- 辅助根因分析
   2. 存储security context
 
 
注意:
    
      所有的消息都是不可修改的(immutable), 去消息中存储数据, 意味着基于之前的消息创建一个新的消息; 
      这样可以保证消息在多线程和分布式环境的并发安全性; 
      
      
### 命令(Commands)      
      
命令用来描述更改程序状态的意图, 通常为CommandMessage接口的实现类;

命令总是有唯一的一个目标, 它不关心哪个组件处理该命令或者是该组件在哪里, 但是可能关心其输出, 所以通常命令消息通过 Command Bus 发送, 并且支持得到一个返回值; 

### 事件(Events)

事件用来描述程序内部发生的事情, 通常的事件源为Aggregate, 在Axon中, 事件可以是所有的对象, 所以强烈建议将所有的事件都支持序列化; 

当事件被派发的时候, 事件被包装成EventMessage, 实际的消息类型将会取决于事件来源
1. 来自Aggregate: DominEventMessage 
    除了所有事件都有的唯一标识, 该事件还包含 
    1. 类型(type) 
    2. 时间戳(timestamp)
    3. sequence number: 发射源发送事件的序号; 
2. 其他: EventMessage
      
注意：
    
    尽管 DominEventMessage 包含一个指向 Aggregate 的引用, 你也应当在Event自身中包含此标识, DominEventMessage 中的标识 可以被EventStore拿来存储事件;
    
    
事件被存储在EventMessage的payload字段中, 另外, 还可以存储 metadata。
metadata 通常用来存储业务不关心的信息, 比如审计信息;


注意:
    
    通常metadata中不要包含 业务信息, 如果依赖了, 通常应该将这些消息放在Event中; 
    

尽管并非强制, 但是通常 领域事件的最佳实践是不可修改, 所以最好将所有的字段生命为 final并且在构造方法中初始化这些字段, 当该事件类型比较复杂时使用 Builder 模式; 


注意:

    尽管领域事件技术上用来表示状态变更, 你也应当通过事件来捕获状态变更意图。最佳实践: 
    抽象类记录状态变更, 实现类表示意图;
    比如抽象的 AddressChangedEvent, 和 其两个实现类： ContactMovedEvent 和 AddressCorrectedEvent, 某些监听器不关心意图, 就可以直接监听抽象类, 关心意图的监听器就可以关心具体的事件;
    
    

### 查询(Queries)
    
查询用来描述状态的请求信息, 可以有多个handlers, 当派发查询事件时,客户端需要标识出其想从一个或者多个可用的查询处理器中获取结果;    
    



## 解剖消息(Anatomy of a Message)

Axon中, 所有的通信都是通过明确消息传递, 这些消息都会实现 Message 接口
消息包含
1. Payload:  标识消息的实际功能; 
2. Meta Data: 描述消息的上下文;

每个消息的子接口代表一种类型的消息, 并且定义额外的信息, 区别于Meta Data, 这些额外信息定义了正确处理该消息所需使用到的信息;


消息都是不可变得. 意味着在消息中添加Meta Data数据, 会触发创建一个新的附带额外信息的消息实例, 但是这两个消息实例仍然只代表同一个概念(conceptual message), 底层是因为每个消息都会有一个唯一的标识, 更改消息的元数据, 并不会更改该标识;


### 元数据(Meta Data)

元数据可以用来标识消息是如何生成的, 比如, Meta Data 可以包含生成该消息的其他消息的信息;

Axon 中, Meta Data 通常用来标识一个 String, Object 为泛型参数的Map, 所以你可以添加任意类型的值, 作为元数据的值, 但是依然建议使用基本类型和字符串。在进行结构修改的时候, 元数据不具有和payload相同的灵活性( 元数据不包含类型信息 ?)

和不同的Map<String, Object>不同的是, MetaData 是不可以修改的, 修改方法将会创建或者是返回一个新的实例,  而不仅仅是修改已经存在的对象; 

```JAVA
MetaData metaData = MetaData.with("myKey", 42) // 1
                            .and("otherKey", "some value"); // 2
```

1. 创建 MetaData 实例;
2. 添加其他键值对, 这时候会返回新的实例;

Messsage 中的MetaData 也是类似的; 

```JAVA

EventMessage eventMessage = 
        GenericEventMessage.asEventMessage("myPayload") // 1
                           .withMetaData(singletonMap("myKey", 42)) // 2
                           .andMetaData(singletonMap("otherKey", "some value")); // 2
```


1. 创建包含 消息, 比payload 有效负载区内容为 "myPayload"
2. withMetaData 使用给定的map替换原本MetaData中的信息
3. addMetaData 新增新的键值对, 如果已经存在相同的key, 其将会被替换;


### 消息相关的数据 (Message-specific data)

确定的消息将会包含额外的信息, 
1. EventMessage 包含时间戳, 标识产生该事件的时间，
2. QueryMessage 除了Payload和Meta data还包含期待返回的数据类型信息; 


## 消息相关(Message Correlation)

在消息系统中， 经常会将消息聚合在一起，或者是关联起来。
Axon框架中的 一个 Command 消息，会产生一个或者多个Event消息，
一个Query消息，会产生一个或者是多个QueryResponse 消息。
通常，关联是通过指定消息的属性实现，一般被称为 关联标识（correlation identifier）

### 关联数据提供器（Correlation Data Provider）

Axon中使用MetaData 属性传输消息的元数据。 为了填充 被 Unit of work 产生的新消息的MetaData，通常会使用到 CorrelationDataProvider。

Axon 当前提供如下几种实现：

#### MessageOriginProvider

这是默认的实现
主要负责将correlationId 和 traceId 传输到另一个消息中

1. correlationId 总是指向产生新消息的父消息的的标识；
2. traceId  总是为开启此次调用链的消息（根消息）的  traceId；

当这两个字段都没有值时，两个字段都会被消息标识填充；
比如，如果接受到一个Command消息，并产生一个Event消息
那么Event消息的MetaData将会这样填充

1. 命令消息的标识为 correlationId
2. 命令消息在MetaData中附带的traceId，或者是消息的唯一标识称为traceId；


#### SimpleCorrelationDataProvider

无条件复制指定的key列表到另外一个Message

比如
```JAVA

public class Configuration {

    public CorrelationDataProvider customCorrelationDataProvider() {
        return new SimpleCorrelationDataProvider("myId", "myId2");
    }
}
```


#### MultiCorrelationDataProvider

组合多个providers的能力

比如

```JAVA

public class Configuration {

    public CorrelationDataProvider customCorrelationDataProviders() {
        return new MultiCorrelationDataProvider<CommandMessage<?>>(
            Arrays.asList(
                new SimpleCorrelationDataProvider("someKey"),
                new MessageOriginProvider()
            )
        );
    }
}
```

#### 实现自定义的关联提供器

如果以上不能满足需求那么可以实现自定义的；

```JAVA
public class AuthCorrelationDataProvider implements CorrelationDataProvider {

    private final Function<String, String> usernameProvider;

    public AuthCorrelationDataProvider(Function<String, String> userProvider) {
        this.usernameProvider = userProvider;
    }

    @Override
    public Map<String, ?> correlationDataFor(Message<?> message) {
        Map<String, Object> correlationData = new HashMap<>();
        if (message instanceof CommandMessage<?>) {
            if (message.getMetaData().containsKey("authorization")) {
                String token = (String) message.getMetaData().get("authorization");
                correlationData.put("username", usernameProvider.apply(token));
            }
        }
        return correlationData;
    }
}
```


如上将会将用户的用户名填充到新的MetaData中;


## Configuration

当默认的MessageOriginProvider不能满足使用，则需将自定义的提供者注册到程序中。

1. 如果使用到了Axon配置API，需要调用其Configuration#configureCorrelationDataProvider

```JAVA
public class Configuration {

    public void configuring() {
        Configurer configurer = 
            DefaultConfigurer.defaultConfiguration()
                             .configureCorrelationDataProviders(config -> Arrays.asList(
                                new SimpleCorrelationDataProvider("someKey"),
                                new MessageOriginProvider()
                             ));
    }
}
```

如果使用SpringBoot的自动配置功能,则只需要提供一个工厂方法，暴露出Spring Bean即可，
下面demo演示注册了多个提供者的写法。

```JAVA
@Configuration
public class CorrelationDataProviderConfiguration {

    // Configuring a single CorrelationDataProvider will automatically override the default MessageOriginProvider
    @Bean
    public CorrelationDataProvider someKeyCorrelationProvider() {
        return new SimpleCorrelationDataProvider("someKey");
    }    

    @Bean
    public CorrelationDataProvider messageOriginProvider() {
        return new MessageOriginProvider();
    }
}
```


## 消息拦截（Message Intercepting）

两种不同类型的拦截器

1. dispatch interceptors
    1. 在消息被派发到消息处理器之前被调用，此时甚至不知道有没有消息处理器存在；
2. handler interceptors
    1. 在消息被消息处理器处理之前调用

### 命令拦截器 （Command Interceptors）

使用command bus的优点是，可以拦截所有的输入命令，并执行一些底层活动。
比如日志或者是登录等这些业务无关的功能。

#### 命令派发拦截器（ Command Dispatch Interception ）

消息派发拦截器在命令被派发到command bus时被调用，
1. 它们拥有通过增加metadata修改命令消息的能力。
2. 通过抛出异常的形式阻塞命令。
3. 总是在派发线程被执行;

比如，下面打印命令消息的拦截器

```JAVA
public class MyCommandDispatchInterceptor implements MessageDispatchInterceptor<CommandMessage<?>> {

    private static final Logger LOGGER = LoggerFactory.getLogger(MyCommandDispatchInterceptor.class);

    @Override
    public BiFunction<Integer, CommandMessage<?>, CommandMessage<?>> handle(List<? extends CommandMessage<?>> messages) {
        return (index, command) -> {
            LOGGER.info("Dispatching a command {}.", command);
            return command;
        };
    }
}
```

如下我如何将该拦截器注册到CommandBus

```JAVA
public class CommandBusConfiguration {

    public CommandBus configureCommandBus() {
        CommandBus commandBus = SimpleCommandBus.builder().build();
        commandBus.registerDispatchInterceptor(new MyCommandDispatchInterceptor());
        return commandBus;
    }
}
```

##### 结构验证(Structural validation)

其实如果命令消息并不包含所有必须的信息或者是格式不正确，那么我们处理它是没有意义的。
所以，其越早被发现越好，最好在一个事务开始之前。 
因此需要一个拦截器去检查输入的所有命令。 

这里成为结构性校验。

Axon支持 JS R303 Bean Validation. 因此你可以使用比如 @NotEmpty 和 @Pattern等这些注解。

1. 你需要在classpath中引入 JSR 303(比如 Hibernate-Validator)。
2. 在Command bus 中配置 BeanValidationInterceptor, 其将会自动查找和配置你的验证器实现


拦截器顺序技巧
    
    原则是尽量话费最少的资源在无效的命令上面, 所以验证拦截器应该仅仅放在LoggingInterceptor 或者 AuditingInterceptor 之后.
    

BeanValidationInterceptor 也实现了MessageHandlerInterceptor接口, 所以也可以配置为handler拦截器；

#### 命令处理拦截器(Command Handler Interceptors)
    

命令处理器拦截器
1. 在命令处理前后被执行
2. 可以中断命令的执行，比如因为安全的原因；

必须实现MessageHandlerInterceptor接口，该接口只有一个方法handle, 包含两个参数
1. UnitOfWork:
    1. 提供当前正在被处理的消息
    1. 提供在逻辑上优先结合的可能性;
        [拓展](https://docs.axoniq.io/reference-guide/axon-framework/messaging-concepts/unit-of-work)
       [ ] ？？？
    
2. InterceptorChain: 
    用于继续推进派发处理;
    

相对派发拦截器来说, 处理器拦截器是在命令被处理的上下文中被执行, 因此，这意味着他们可以根据正在处理的消息将相关性数据附加到工作单元。然后，此相关性数据将附加到在该工作单元的上下文中创建的消息中.


处理器拦截器也可以被用来管理事务。此时，我们可以注册 TransactionManagingInterceptor，然后配置TransactionManager，后者用来开启、提交、回滚底层的事务;


下面通过创建一个消息拦截器去验证如果MetaData中是否包含 userId 字段，并且其值为axonUser。
如果没有改字段将会抛出异常，如果值不匹配我们也不会继续处理

```java

public class MyCommandHandlerInterceptor implements MessageHandlerInterceptor<CommandMessage<?>> {

    @Override
    public Object handle(UnitOfWork<? extends CommandMessage<?>> unitOfWork, InterceptorChain interceptorChain) throws Exception {
        CommandMessage<?> command = unitOfWork.getMessage();
        String userId = Optional.ofNullable(command.getMetaData().get("userId"))
                                .map(uId -> (String) uId)
                                .orElseThrow(IllegalCommandException::new);
        if ("axonUser".equals(userId)) {
            return interceptorChain.proceed();
        }
        return null;
    }
}
```


下面将注册该处理器来节气到CommandBus:

```JAVA
public class CommandBusConfiguration {

    public CommandBus configureCommandBus() {
        CommandBus commandBus = SimpleCommandBus.builder().build();
        commandBus.registerHandlerInterceptor(new MyCommandHandlerInterceptor());
        return commandBus;
    }

}
```


#### @CommandHandlerInterceptor 注解


框架支持将 Aggregate/Entity 中被 @CommandHandlerInterceptor 注解的方法添加为命令拦截器. 和正常的拦截器不同的该拦截器可以使用 当前 Aggregate的属性作为决策参数.

[ ] TODO 

* The annotation can be put on entities within the Aggregate.
* It is possible to intercept a command on Aggregate Root level, whilst the command handler is in a child entity.
* Command execution can be prevented by firing an exception from an annotated command handler interceptor.
* It is possible to define an InterceptorChain as a parameter of the command handler interceptor method and use it to control command execution.
* By using the commandNamePattern attribute of the @CommandHandlerInterceptor annotation we can intercept all commands matching the provided regular expression.
* Events can be applied from an annotated command handler interceptor.

下面的例子中我们可以看到 一个被@CommandHandlerInterceptor注解的方法,
在命令中的状态和Aggregate中的状态不一致的时候 会拒绝执行; 


```JAVA
public class GiftCard {
    //..
    private String state;
    //..
    @CommandHandlerInterceptor
    public void intercept(RedeemCardCommand command, InterceptorChain interceptorChain) {
        if (this.state.equals(command.getState())) {
            interceptorChain.proceed();
        }
    }
}
```

@CommandHandlerInterceptor 本质上是 @MessageHandlerInterceptor 的具体实现 [more](https://docs.axoniq.io/reference-guide/axon-framework/messaging-concepts/message-intercepting#messagehandlerinterceptor)


## 事件拦截器(Event Interceptors)

和命令拦截器一样事件拦截器也可以通过派发拦截器和处理器拦截器在发布和执行前后被拦截

### 事件拦截器（Event Dispatch Interceptors）

任何被注册入EventBus的派发拦截器都会在事件被发布后执行, 它们可以通过增加metadata修改message. 
同样在发布事件的线程执行；

创建派发事件拦截器使得将发布的事件通过日志输出；

```JAVA
public class EventLoggingDispatchInterceptor
                implements MessageDispatchInterceptor<EventMessage<?>> {

    private static final Logger logger =
                LoggerFactory.getLogger(EventLoggingDispatchInterceptor.class);

    @Override
    public BiFunction<Integer, EventMessage<?>, EventMessage<?>> handle(
                List<? extends EventMessage<?>> messages) {
        return (index, event) -> {
            logger.info("Publishing event: [{}].", event);
            return event;
        };
    }
}
```

下面将该拦截器注册到EventBus

```JAVA
public class EventBusConfiguration {

    public EventBus configureEventBus(EventStorageEngine eventStorageEngine) {
        // note that an EventStore is a more specific implementation of an EventBus
        EventBus eventBus = EmbeddedEventStore.builder()
                                              .storageEngine(eventStorageEngine)
                                              .build();
        eventBus.registerDispatchInterceptor(new EventLoggingDispatchInterceptor());
        return eventBus;
    }
}
```


### 事件处理拦截器（Event Handler Interceptors）

消息处理器可以在事件前后触发执行动作。也可以通过异常的性格是阻塞事件的执行，比如因为安全原因。

拦截器通常需要实现 MessageHandlerInterceptor 接口. 
这个接口只包含 handle() 一个方法，和两个参数 UnitOfWork 、InterceptorChain


    这里和命令拦截器相似, 省略...


```JAVA
public class MyEventHandlerInterceptor
        implements MessageHandlerInterceptor<EventMessage<?>> {

    @Override
    public Object handle(UnitOfWork<? extends EventMessage<?>> unitOfWork,
                         InterceptorChain interceptorChain) throws Exception {
        EventMessage<?> event = unitOfWork.getMessage();
        String userId = Optional.ofNullable(event.getMetaData().get("userId"))
                                .map(uId -> (String) uId)
                                .orElseThrow(IllegalEventException::new);
        if ("axonUser".equals(userId)) {
            return interceptorChain.proceed();
        }
        return null;
    }
}
```


### 查询拦截器（Query Interceptors）


### @MessageHandlerInterceptor

### @ExceptionHandler

仅仅当出现异常的结果时，才会被调用, 

 
```JAVA
public class CardSummaryProjection {

    /*
     * Some @EventHandler and @QueryHandler annotated methods
     */
    @ExceptionHandler
    public void handle(Exception exception) {
        // How you prefer to react to this generic exception,
        //  for example by throwing a domain specific exception.
    }

    @ExceptionHandler(resultType = IllegalArgumentException.class)
    public void handle(IllegalArgumentException exception) {
        // How you prefer to react to the IllegalArgumentException,
        //  for example by throwing a domain specific exception.
    }
}
```

## Unit of Work


![0048c7246cd5547eae78bebddc944404.png](evernotecid://0C0C6CA7-E0B1-4D07-A08B-2457E22E1166/appyinxiangcom/2181761/ENResource/p481)

UnitOfWork 是Axon框架中的重要概念. 尽管大部分情况下我们不会直接操作它，如果把消息的处理看做是一个工作单元，那么工作单元的目的就是协调消息处理中涉及的多个行为。因此各个组件可以注册UnitOfWork的各种actions, 比如onPrepareCommit 或者是onCleanUp.


通常情况下我们不需要直接访问UnitOfWork, 通常情况下我们通过使用Axon框架提供的构建块(building blocks)使用之, 如果确实要使用的话，可以有如下几种方式
1. handler method 方法参数: 如果该方法是注解方法，那么直接放类型为 UnitOfWork的类型的参数即可;
2. 通过线程上下文CurrentUnitOfWork.get() 获取;


需要获取UnitOfWork的原因有我们需要绑定(attach)一些可重用得到资源，使得在该消息处理的过程中可以随时获取，或者是当工作完成后需要清理之;

在这种情况下, unitOfWork.getOrComputeResource() 和 生命周期回调方法
比如 OnRollback() afterCommit() 和 onCleanUp() 就可以允许我们注册资源和生命actions。

注意：
    
    UnitOfWork 可以理解为一种修改缓存，并不是事务的替代品。尽管所有的状态变更都是在其被提交的的时候被提交，它的提交也并不是原子的。最佳实践是一个命令尽量不要超过多个action，如果超过多个ation那么需要在UnitOfWork的commit阶段嵌入事务。后者我们可以通过 unitOfWork.OnCommit(..) 函数去注册事务提交函数，该函数会在UnitOfWork提交的时候被回调处理.
    

在handler中可能会抛出异常，默认情况下未检查异常将会触发UnitOfWork回滚所有的变更。并期望最后消除所有副作用。

Axon提供一些如下一些回滚策略：

1. RollbackConfigurationType.NEVER
    总是提交
2. RollbackConfigurationType.ANY_THROWABLE
    任何异常发生都会回滚
3. RollbackConfigurationType.UNCHECKED_EXCEPTIONS
    在errors或者是runtime异常回滚
4. RollbackConfigurationType.RUNTIME_EXCEPTION
    运行时异常回滚(error不会)
    

    
当使用框架中的组件处理消息是，UnitOfWork将会自动管理生命周期。但是如果我们不使用它，需要自己处理：需要自行编程实现start和commit(或者是roll back)

大部分使用场景DefaultUnitOfWork将会提供你想要的所有功能，它通常处理一个单线程任务。如果想在UnitOfWork中执行一个任务可以这样操作

1. UnitOfWork.execute(Runnable)
2. UnitOfWork.executeWithResult(Callable)

上面都会创建一个新的DefaultUnitOfWork.

UnitOfWork会依次被启动 并在任务完成时被提交或者是任务失败时被回滚. 你可以选择手动启动，提交或者是回滚工作单元. 

经典用法如下:

```java
UnitOfWork uow = DefaultUnitOfWork.startAndGet(message);
// then, either use the autocommit approach:
uow.executeWithResult(() -> ... logic here);

// or manually commit or rollback:
try {
    // business logic comes here
    uow.commit();
} catch (Exception e) {
    uow.rollback(e);
    // maybe rethrow...
}
```

Note:

   UnitOfWork 运行总是围绕者消息
   1. 开始于某个即将被处理的消息.
   2. 执行的结果 





