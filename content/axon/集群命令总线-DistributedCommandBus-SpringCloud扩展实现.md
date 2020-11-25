---
title: "集群命令总线 DistributedCommandBus SpringCloud扩展实现"
date: 2020-11-25T23:36:25+08:00
draft: false
---


集群命令总线用于消息在多个节点之间的转发, SpringCloud扩展实现使用一致性hash算法, 有效的将消息的压力负载到多个节点上面;


![ss](https://cdn.jsdelivr.net/gh/halley-eng/halley/static/images/axon/2_1.jpg)

集群命令总线相对于简单命令总线多了路由器和连接器;

#### 消息派发过程

1. 通过路由器找到目标节点 destination;
    1. 使用连接器发送出去
2. 找不到目标节点则抛出异常;

```java
    @Override
    public <C> void dispatch(CommandMessage<C> command) {
        if (defaultCommandCallback != null) {
            dispatch(command, defaultCommandCallback);
            return;
        }

        LoggingCallback loggingCallback = LoggingCallback.INSTANCE;
        // 没有消息监控器 则直接派发;
        if (NoOpMessageMonitor.INSTANCE.equals(messageMonitor)) {
            // 派发拦截器
            CommandMessage<? extends C> interceptedCommand = intercept(command);
            // 路由找节点;
            Optional<Member> optionalDestination = commandRouter.findDestination(interceptedCommand);
            if (optionalDestination.isPresent()) {
                Member destination = optionalDestination.get();
                try {
                    // 发送到目标节点;
                    connector.send(destination, interceptedCommand);
                } catch (Exception e) {
                    destination.suspect();
                    loggingCallback.onResult(interceptedCommand, asCommandResultMessage(
                            new CommandDispatchException(DISPATCH_ERROR_MESSAGE + ": " + e.getMessage(), e)
                    ));
                }
            } else {
                loggingCallback.onResult(interceptedCommand, asCommandResultMessage(new NoHandlerForCommandException(
                        format("No node known to accept [%s]", interceptedCommand.getCommandName())
                )));
            }
        } else {
            dispatch(command, loggingCallback);
        }
    }

```

### SpringCloud 路由器 (SpringCloudCommandRouter) -- 找节点
    
![ss](https://cdn.jsdelivr.net/gh/halley-eng/halley/static/images/axon/2_2.jpg

节点路由是路由键根据路由规则去路由表匹配出节点的过程.
所以我们先说明下如何构建一致性hash路由表;

#### 从服务发现中心本地化服务地址和路由信息到新的一致性hash

```JAVA
    public void updateMemberships(HeartbeatEvent event) {
        updateMemberships();
    }

    private void updateMemberships() {
        AtomicReference<ConsistentHash> updatedConsistentHash = new AtomicReference<>(new ConsistentHash());
        // 服务发现节点列表;
        List<ServiceInstance> instances = discoveryClient.getServices().stream()
                                                         .map(discoveryClient::getInstances)
                                                         .flatMap(Collection::stream)
                                                         .filter(serviceInstanceFilter)
                                                         .collect(Collectors.toList());
        // 将黑名单部分删除;
        cleanBlackList(instances);
        // 将远程的路由信息更新到一致性hash;
        instances.stream()
                 .filter(this::ifNotBlackListed)
                 .forEach(serviceInstance -> updateMembershipForServiceInstance(serviceInstance,
                                                                                updatedConsistentHash));
        // 更新后的一致性hash重新设置到本地;
        ConsistentHash newConsistentHash = updatedConsistentHash.get();
        atomicConsistentHash.set(newConsistentHash);
        // 通知相关监听器hash表已经变更;
        consistentHashChangeListener.onConsistentHashChanged(newConsistentHash);
    }
```

从节点信息中提取路由信息, 并重建一致性hash表的过程如下:

```JAVA
    private Optional<ConsistentHash> updateMembershipForServiceInstance(ServiceInstance serviceInstance,
                                                                        AtomicReference<ConsistentHash> atomicConsistentHash) {
        if (logger.isDebugEnabled()) {
            logger.debug("Updating membership for service instance: [{}]", serviceInstance);
        }
        // 构建节点;
        Member member = buildMember(serviceInstance);
        // 获取路由信息;
        Optional<MessageRoutingInformation> optionalMessageRoutingInfo = getMessageRoutingInformation(serviceInstance);
        // 将路由信息更新到节点;
        if (optionalMessageRoutingInfo.isPresent()) {
            MessageRoutingInformation messageRoutingInfo = optionalMessageRoutingInfo.get();
            // 将路由节点信息 更新到一致性hash (添加新的节点)
            return Optional.of(atomicConsistentHash.updateAndGet(
                    consistentHash -> consistentHash.with(member,
                                                          messageRoutingInfo.getLoadFactor(),
                                                          // 将原始命令过滤器在反序列化过来;
                                                          messageRoutingInfo.getCommandFilter(serializer))
            ));
        } else {
            logger.info(
                    "Black listed ServiceInstance [{}] under host [{}] and port [{}] since we could not retrieve the "
                            + "required Message Routing Information from it.",
                    serviceInstance.getServiceId(), serviceInstance.getHost(), serviceInstance.getPort()
            );
            blackListedServiceInstances.add(serviceInstance);
        }
        return Optional.empty();
    }

```

然后就可以从路由表中获取节点信息; 

#### 根据路由信息查询目标节点

```JAVA
    @Override
    public Optional<Member> findDestination(CommandMessage<?> commandMessage) {
        return atomicConsistentHash.get().getMember(routingStrategy.getRoutingKey(commandMessage), commandMessage);
    }

```

如上实际执行过程中
1. 通过路由策略拿到路由键
![ss](https://cdn.jsdelivr.net/gh/halley-eng/halley/static/images/axon/2_3.jpg)
2. 根据路由键去一致性hash表(ConsistentHash)，获取实际的下一步地址; 
        
    1. 有序map中查询大于当前key hash值的节点列表;
    2. 从上述列表中过滤出来第一个member 

    ```JAVA
    public Optional<Member> getMember(String routingKey, CommandMessage<?> commandMessage) {
        String hash = hash(routingKey);
        Optional<Member> foundMember = this.findSuitableMember(commandMessage, this.hashToMember.tailMap(hash).values());
        if (!foundMember.isPresent()) {
            foundMember = this.findSuitableMember(commandMessage, this.hashToMember.headMap(hash).values());
        }

        return foundMember;
    }

    private Optional<Member> findSuitableMember(CommandMessage<?> commandMessage, Collection<ConsistentHash.ConsistentHashMember> members) {
        Stream var10000 = members.stream().filter((member) -> {
            return member.commandFilter.matches(commandMessage);
        });
        Member.class.getClass();
        return var10000.map(Member.class::cast).findAny();
    }
    ```



### SpringCloud 连接器  -- 发送数据包

![ss](https://cdn.jsdelivr.net/gh/halley-eng/halley/static/images/axon/2_4.jpg)

由Builder可知该链接器依赖外部资源如下
```JAVA
    
        // 1. 本地命令总线, 用于将接收到的远程命令发布到本地总线执行;
        private CommandBus localCommandBus;
        // 2. 发布本地消息到远程节点;
        private RestOperations restOperations;
        // 3. 节点之间的消息传输序列化; 
        private Serializer serializer;
        // 4. 异步执行将消息发送到其他节点的过程
        private Executor executor = DirectExecutor.INSTANCE;
```


#### 发送消息到远程节点; 


```JAVA
    @Override
    public <C> void send(Member destination, CommandMessage<? extends C> commandMessage) {
        shutdownLatch.ifShuttingDown("JGroupsConnector is shutting down, no new commands will be sent.");
        // 1. 本地节点直接派发;
        if (destination.local()) {
            localCommandBus.dispatch(commandMessage);
        // 2. 远程节点发送到远程服务器;             
        } else {
            executor.execute(() -> sendRemotely(destination, commandMessage, DO_NOT_EXPECT_REPLY));
        }
    }
```

具体发送方法如下：

```JAVA
    private <C, R> ResponseEntity<SpringHttpReplyMessage<R>> sendRemotely(Member destination,
                                                                          CommandMessage<? extends C> commandMessage,
                                                                          boolean expectReply) {
        Optional<URI> optionalEndpoint = destination.getConnectionEndpoint(URI.class);
        if (optionalEndpoint.isPresent()) {
            // 构建远程地址
            URI endpointUri = optionalEndpoint.get();
            URI destinationUri = buildURIForPath(endpointUri.getScheme(), endpointUri.getUserInfo(),
                                                 endpointUri.getHost(), endpointUri.getPort(), endpointUri.getPath());
            // 封装HTTP消息包
            SpringHttpDispatchMessage<C> dispatchMessage =
                    new SpringHttpDispatchMessage<>(commandMessage, serializer, expectReply);
            // 发送HTTP消息包
            return restOperations.exchange(destinationUri,
                                           HttpMethod.POST,
                                           new HttpEntity<>(dispatchMessage),
                                           new ParameterizedTypeReference<SpringHttpReplyMessage<R>>() {
                                           });
        } else {
            String errorMessage = String.format("No Connection Endpoint found in Member [%s] for protocol [%s] " +
                                                        "to send the command message [%s] to",
                                                destination, URI.class, commandMessage);
            logger.error(errorMessage);
            throw new IllegalArgumentException(errorMessage);
        }
    }
```

注意以上消息的发布过程是同步的, 实际参见如下接口实现:

RestTemplate#exchange


#### 接收远程消息名分发到本地消息总线

```JAVA
    @PostMapping("/command")
    public <C, R> CompletableFuture<?> receiveCommand(@RequestBody SpringHttpDispatchMessage<C> dispatchMessage) {

        CommandMessage<C> commandMessage = dispatchMessage.getCommandMessage(serializer);
        // 同步消息, 派发过程使用HTTP消息回调;
        if (dispatchMessage.isExpectReply()) {
            try {
                SpringHttpReplyFutureCallback<C, R> replyFutureCallback = new SpringHttpReplyFutureCallback<>();
                localCommandBus.dispatch(commandMessage, replyFutureCallback);
                return replyFutureCallback;
            } catch (Exception e) {
                logger.error("Could not dispatch command", e);
                return CompletableFuture.completedFuture(createReply(commandMessage, asCommandResultMessage(e)));
            }
        // 不期望回复的情况, 仅仅丢进bus执行;
        } else {
            try {
                localCommandBus.dispatch(commandMessage);
                return CompletableFuture.completedFuture("");
            } catch (Exception e) {
                logger.error("Could not dispatch command", e);
                return CompletableFuture.completedFuture(createReply(commandMessage, asCommandResultMessage(e)));
            }
        }
    }

```

### 消息发送和响应协议报文

#### 发送

![ss](https://cdn.jsdelivr.net/gh/halley-eng/halley/static/images/axon/2_5.jpg)


#### 回复

![ss](https://cdn.jsdelivr.net/gh/halley-eng/halley/static/images/axon/2_6.jpg)


### 总结

本文分析了 分布式命令总线的SpringCloud扩展实现SpringHttpCommandBusConnector的源码
包含
1. 根据节点信息构建路由表
2. 根据一致性hash路由表给消息路由
3. 通过连接器将消息发送到三方节点
4. 通过HTTP消息接收命令，并转发到本地Bus执行; 

不包含内容
1. 路由键解析策略
    1. AnnotationRoutingStrategy
    2. MetaDataRoutingStrategy
2. 路由表如何更新
    1. 和心跳的关系;
3. SpringCloud扩展包自动化配置和启动器;
