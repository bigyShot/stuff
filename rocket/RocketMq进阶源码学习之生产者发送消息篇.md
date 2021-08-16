**RocketMq进阶源码学习之生产者发送消息篇**

在RocketMq的生产端这块,最重要的自然是发送消息了,生产者发送消息有同步/异步/单向3种模式,每种模式的处理方式也都也都各不相同,最重要的是同步/异步的处理方式,单向应用场景较少(一般适用于对消息可靠性要求不高的场景,如发送日志),本文将主要分析同步/异步.

老规矩,从实例开始,一个最简单的发送消息代码

```java
DefaultMQProducer producer = new DefaultMQProducer("ProducerGroupName");
producer.start();
Message msg=new Message();
//同步发送
SendResult sendResult = producer.send(msg);
// 异步发送消息, 发送结果通过callback返回给客户端。
producer.sendAsync(msg, new SendCallback() {
    @Override
    public void onSuccess(final SendResult sendResult) {
        // 消息发送成功。
        System.out.println("send message success. topic=" + sendResult.getTopic() + ", msgId=" + sendResult.getMessageId());
    }

    @Override
    public void onException(OnExceptionContext context) {
        // 消息发送失败，需要进行重试处理，可重新发送这条消息或持久化这条数据进行补偿处理。
        System.out.println("send message failed. topic=" + context.getTopic() + ", msgId=" + context.getMessageId());
    }
});
```

先看同步的时候,进入send方法,经过N个send的链式调用,进入真正的逻辑处理方法.再回头看异步发送的时候,其实除了一开始传入一个SendCallback,在前几个链式调用中明确CommunicationMode是Async之外,最终都是进入接下来的这个方法,无论是Async/Sync/Oneway都统一在这里处理

这里的处理是
1.先检查producer是否已经启动
2.再校验message的内容是否合规(比如Topic是否为空,Topic字符是否合法等等,这个字符校验上我就犯过一次错,Topic的内容不能含有空格,切记检查清楚配置文件的brokerClusterName,在回应request消息的时候会默认设置Topic的值上包含clusterName)
3.根据Topic的值去查找Topic的路由信息
4.检查发送模式,如果是同步则只发送一次,异步则会在发送失败时进行重试,最多尝试配置的重试次数+1
5.从Topic的多个MessageQueue中选中一个发送消息

```java
    private SendResult sendDefaultImpl(
        Message msg,
        final CommunicationMode communicationMode,
        final SendCallback sendCallback/**sendCallback从头贯彻到尾,是为了接收真正发送消息的结果 */,
        final long timeout
    ) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
        /** 检查producer是否已经启动*/
        this.makeSureStateOK();
        /** 校验一下message的内容*/
        Validators.checkMessage(msg, this.defaultMQProducer);
        final long invokeID = random.nextLong();
        long beginTimestampFirst = System.currentTimeMillis();
        long beginTimestampPrev = beginTimestampFirst;
        long endTimestamp = beginTimestampFirst;
        /** 查找topic的信息*/
        TopicPublishInfo topicPublishInfo = this.tryToFindTopicPublishInfo(msg.getTopic());
        if (topicPublishInfo != null && topicPublishInfo.ok()) {
            boolean callTimeout = false;
            MessageQueue mq = null;
            Exception exception = null;
            SendResult sendResult = null;
            /**异步就最多可以重试1+配置的重试次数,同步则最多只尝试一次*/
            int timesTotal = communicationMode == CommunicationMode.SYNC ? 1 + this.defaultMQProducer.getRetryTimesWhenSendFailed() : 1;
            int times = 0;
            /**记录尝试发送过哪些broker*/
            String[] brokersSent = new String[timesTotal];
            for (; times < timesTotal; times++) {
                String lastBrokerName = null == mq ? null : mq.getBrokerName();
                /**所以发送消息的负载均衡实在客户端实现的,默认是从Topic的所有的messageQueue中指定一个
                 * 这里使用了ThreadLocal,将messageQueue的index存在threadLocal中,这样之后如果重试消息,
                 * 还能拿到之前的index,然后基于之前的选择的messageQueue重新选择,默认是每次index+1*/
                MessageQueue mqSelected = this.selectOneMessageQueue(topicPublishInfo, lastBrokerName);
                if (mqSelected != null) {
                    mq = mqSelected;
                    brokersSent[times] = mq.getBrokerName();
                    try {
                        beginTimestampPrev = System.currentTimeMillis();
                        if (times > 0) {
                            //Reset topic with namespace during resend.
                      msg.setTopic(this.defaultMQProducer.withNamespace(msg.getTopic()));
                        }
                        long costTime = beginTimestampPrev - beginTimestampFirst;
                        if (timeout < costTime) {
                            callTimeout = true;
                            break;
                        }

                        sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, topicPublishInfo, timeout - costTime);
                        endTimestamp = System.currentTimeMillis();
                        this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false);
                        switch (communicationMode) {
                            case ASYNC:
                                return null;
                            case ONEWAY:
                                return null;
                            case SYNC:
                                if (sendResult.getSendStatus() != SendStatus.SEND_OK) {
                                    if (this.defaultMQProducer.isRetryAnotherBrokerWhenNotStoreOK()) {
                                        continue;
                                    }
                                }

                                return sendResult;
                            default:
                                break;
                        }
                    } catch (RemotingException e) {
                .........删掉了很多对于exception的处理,对于此处的源码理解无意义
            if (sendResult != null) {
                return sendResult;
            }

            String info = String.format("Send [%d] times, still failed, cost [%d]ms, Topic: %s, BrokersSent: %s",
                times,
                System.currentTimeMillis() - beginTimestampFirst,
                msg.getTopic(),
                Arrays.toString(brokersSent));

            info += FAQUrl.suggestTodo(FAQUrl.SEND_MSG_FAILED);

            MQClientException mqClientException = new MQClientException(info, exception);
            if (callTimeout) {
                throw new RemotingTooMuchRequestException("sendDefaultImpl call timeout");
            }

        validateNameServerSetting();

        throw new MQClientException("No route info of this topic: " + msg.getTopic() + FAQUrl.suggestTodo(FAQUrl.NO_TOPIC_ROUTE_INFO),
            null).setResponseCode(ClientErrorCode.NOT_FOUND_TOPIC_EXCEPTION);
    }
```

这里继续跟踪sendKernelImpl方法,从方法名看,这是发送消息的内核实现

1.根据上面传下来的MessageQueue查找broker的ip,如果找不到这个broker的ip就重新另选一个MessageQueue(其实broker还有一个vip端口,挺有意思的)ps:果然哪里都有VIP啊:)
2.处理消息压缩(异步发送的对情况下会对压缩做特殊的处理,看下面注释)
3.判断是否是事务消息
4.执行钩子函数
5.组装请求头
6.根据消息发送模式选择不同的发送策略

```java
    private SendResult sendKernelImpl(final Message msg,
        final MessageQueue mq,
        final CommunicationMode communicationMode,
        final SendCallback sendCallback,
        final TopicPublishInfo topicPublishInfo,
        final long timeout) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
        long beginStartTime = System.currentTimeMillis();
        String brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());
        /**根据brokerName找到broker的ip地址,如果这个messageQueue的broker找不到就
         * 重新换一个messageQueue,再找broker地址*/
        if (null == brokerAddr) {
            tryToFindTopicPublishInfo(mq.getTopic());
            brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());
        }

        SendMessageContext context = null;
        if (brokerAddr != null) {
            /**broker在启动的时候会启动两个端口监听,一个是默认port10911,一个是port-2
             * 也就是vip端口,这里在这里检测是否要发往vip端口,将
             * vip端口只接收生产者发送请求,不接收消费者的拉取
             * broker监听两个端口的目的是默认端口承载所有网络请求,如果有时候请求非常繁忙,broker端所有I/O线程都在忙
             * 导致后续网络请求进入队列,从而导致消息请求执行缓慢,这种情况下就可以选择将消息发送到vip端口*/
            brokerAddr = MixAll.brokerVIPChannel(this.defaultMQProducer.isSendMessageWithVIPChannel(), brokerAddr);
            byte[] prevBody = msg.getBody();
            try {
                //for MessageBatch,ID has been set in the generating process
                if (!(msg instanceof MessageBatch)) {
                    MessageClientIDSetter.setUniqID(msg);
                }

                boolean topicWithNamespace = false;
                if (null != this.mQClientFactory.getClientConfig().getNamespace()) {
                    msg.setInstanceId(this.mQClientFactory.getClientConfig().getNamespace());
                    topicWithNamespace = true;
                }

                int sysFlag = 0;
                /**压缩消息
                 * 除非是batchMsg或者压缩的时候报错了,不然百分百会压缩成功*/
                boolean msgBodyCompressed = false;
                if (this.tryToCompressMessage(msg)) {
                    sysFlag |= MessageSysFlag.COMPRESSED_FLAG;
                    msgBodyCompressed = true;

                }
                /**判断是否是事务消息*/
                final String tranMsg = msg.getProperty(MessageConst.PROPERTY_TRANSACTION_PREPARED);
                if (tranMsg != null && Boolean.parseBoolean(tranMsg)) {
                    sysFlag |= MessageSysFlag.TRANSACTION_PREPARED_TYPE;
                }
                /**扩展钩子接口,用户可以自定义扩展接口*/
                if (hasCheckForbiddenHook()) {
                    CheckForbiddenContext checkForbiddenContext = new CheckForbiddenContext();
                    checkForbiddenContext.setNameSrvAddr(this.defaultMQProducer.getNamesrvAddr());
                    checkForbiddenContext.setGroup(this.defaultMQProducer.getProducerGroup());
                    checkForbiddenContext.setCommunicationMode(communicationMode);
                    checkForbiddenContext.setBrokerAddr(brokerAddr);
                    checkForbiddenContext.setMessage(msg);
                    checkForbiddenContext.setMq(mq);
                    checkForbiddenContext.setUnitMode(this.isUnitMode());
                    this.executeCheckForbiddenHook(checkForbiddenContext);
                }
                /**扩展钩子接口,在发送消息之前执行*/
                if (this.hasSendMessageHook()) {
                    context = new SendMessageContext();
                    context.setProducer(this);
                    context.setProducerGroup(this.defaultMQProducer.getProducerGroup());
                    context.setCommunicationMode(communicationMode);
                    context.setBornHost(this.defaultMQProducer.getClientIP());
                    context.setBrokerAddr(brokerAddr);
                    context.setMessage(msg);
                    context.setMq(mq);
                    context.setNamespace(this.defaultMQProducer.getNamespace());
                    String isTrans = msg.getProperty(MessageConst.PROPERTY_TRANSACTION_PREPARED);
                    if (isTrans != null && isTrans.equals("true")) {
                        context.setMsgType(MessageType.Trans_Msg_Half);
                    }

                    if (msg.getProperty("__STARTDELIVERTIME") != null || msg.getProperty(MessageConst.PROPERTY_DELAY_TIME_LEVEL) != null) {
                        context.setMsgType(MessageType.Delay_Msg);
                    }
                    this.executeSendMessageHookBefore(context);
                }
                /**组装发送消息的请求头*/
                SendMessageRequestHeader requestHeader = new SendMessageRequestHeader();
                requestHeader.setProducerGroup(this.defaultMQProducer.getProducerGroup());
                requestHeader.setTopic(msg.getTopic());
                requestHeader.setDefaultTopic(this.defaultMQProducer.getCreateTopicKey());
                requestHeader.setDefaultTopicQueueNums(this.defaultMQProducer.getDefaultTopicQueueNums());
                requestHeader.setQueueId(mq.getQueueId());
                requestHeader.setSysFlag(sysFlag);
                requestHeader.setBornTimestamp(System.currentTimeMillis());
                requestHeader.setFlag(msg.getFlag());
                requestHeader.setProperties(MessageDecoder.messageProperties2String(msg.getProperties()));
                requestHeader.setReconsumeTimes(0);
                requestHeader.setUnitMode(this.isUnitMode());
                requestHeader.setBatch(msg instanceof MessageBatch);
                /**如果是重试topic,设置消息重试次数相关属性*/
                if (requestHeader.getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
                    String reconsumeTimes = MessageAccessor.getReconsumeTime(msg);
                    if (reconsumeTimes != null) {
                        requestHeader.setReconsumeTimes(Integer.valueOf(reconsumeTimes));
                        MessageAccessor.clearProperty(msg, MessageConst.PROPERTY_RECONSUME_TIME);
                    }

                    String maxReconsumeTimes = MessageAccessor.getMaxReconsumeTimes(msg);
                    if (maxReconsumeTimes != null) {
                        requestHeader.setMaxReconsumeTimes(Integer.valueOf(maxReconsumeTimes));
                        MessageAccessor.clearProperty(msg, MessageConst.PROPERTY_MAX_RECONSUME_TIMES);
                    }
                }

                SendResult sendResult = null;
                /**
                 * 如果是异步发送就会对压缩消息做点特殊处理,同步不会
                 */
                switch (communicationMode) {
                    case ASYNC:
                        Message tmpMessage = msg;
                        boolean messageCloned = false;
                        /** 如果消息被压缩过,那就将msgBody设置为压缩之前的body,
                         * 使用clone的消息进行发送,防止消息重试消费时被反复压缩*/
                        if (msgBodyCompressed) {
                            //If msg body was compressed, msgbody should be reset using prevBody.
                            //Clone new message using commpressed message body and recover origin massage.
                            //Fix bug:https://github.com/apache/rocketmq-externals/issues/66
                            /** 真正发送的还是压缩过后的消息,只是克隆出来的压缩过后的消息
                             * 克隆之后在把原始消息的body设置为未压缩的状态,那重试的时候再执行压缩
                             * 就不会出现反复压缩的问题了*/
                            tmpMessage = MessageAccessor.cloneMessage(msg);
                            messageCloned = true;
                            msg.setBody(prevBody);
                        }

                        if (topicWithNamespace) {
                            if (!messageCloned) {
                                tmpMessage = MessageAccessor.cloneMessage(msg);
                                messageCloned = true;
                            }
                            msg.setTopic(NamespaceUtil.withoutNamespace(msg.getTopic(), this.defaultMQProducer.getNamespace()));
                        }
                        /**计算是否超时*/
                        long costTimeAsync = System.currentTimeMillis() - beginStartTime;
                        if (timeout < costTimeAsync) {
                            throw new RemotingTooMuchRequestException("sendKernelImpl call timeout");
                        }
                        sendResult = this.mQClientFactory.getMQClientAPIImpl().sendMessage(
                            brokerAddr,
                            mq.getBrokerName(),
                            tmpMessage,
                            requestHeader,
                            timeout - costTimeAsync,
                            communicationMode,
                            sendCallback,
                            topicPublishInfo,
                            this.mQClientFactory,
                            this.defaultMQProducer.getRetryTimesWhenSendAsyncFailed(),
                            context,
                            this);
                        break;
                    case ONEWAY:
                    case SYNC:
                        long costTimeSync = System.currentTimeMillis() - beginStartTime;
                        if (timeout < costTimeSync) {
                            throw new RemotingTooMuchRequestException("sendKernelImpl call timeout");
                        }
                        sendResult = this.mQClientFactory.getMQClientAPIImpl().sendMessage(
                            brokerAddr,
                            mq.getBrokerName(),
                            msg,
                            requestHeader,
                            timeout - costTimeSync,
                            communicationMode,
                            context,
                            this);
                        break;
                    default:
                        assert false;
                        break;
                }
                /**执行消息发送后钩子*/
                if (this.hasSendMessageHook()) {
                    context.setSendResult(sendResult);
                    this.executeSendMessageHookAfter(context);
                }

                return sendResult;
                //老规矩,删除异常处理
            } catch (RemotingException e) {
            } catch (InterruptedException e) {
            } finally {
                msg.setBody(prevBody);
                msg.setTopic(NamespaceUtil.withoutNamespace(msg.getTopic(), this.defaultMQProducer.getNamespace()));
            }
        }
```

还没到,咱还得继续深入,继续追踪sendMessage方法,这里的处理就相对之前比较简单了

1.看能否能拿到消息的"类型"属性,看是否为"REPLY"类型,或者又是否为BatchMessage
2.不同的"类型"构造不同的通信请求
3.根据消息发送模式选择不同的调用

```java
public SendResult sendMessage(
    final String addr,
    final String brokerName,
    final Message msg,
    final SendMessageRequestHeader requestHeader,
    final long timeoutMillis,
    final CommunicationMode communicationMode,
    final SendCallback sendCallback,
    final TopicPublishInfo topicPublishInfo,
    final MQClientInstance instance,
    final int retryTimesWhenSendFailed,
    final SendMessageContext context,
    final DefaultMQProducerImpl producer
) throws RemotingException, MQBrokerException, InterruptedException {
    long beginStartTime = System.currentTimeMillis();
    RemotingCommand request = null;
    String msgType = msg.getProperty(MessageConst.PROPERTY_MESSAGE_TYPE);
    boolean isReply = msgType != null && msgType.equals(MixAll.REPLY_MESSAGE_FLAG);
    if (isReply) {
        if (sendSmartMsg) {
            //该类的字段全为a,b,c...,目的是为了加速fastjson序列化
            SendMessageRequestHeaderV2 requestHeaderV2 = SendMessageRequestHeaderV2.createSendMessageRequestHeaderV2(requestHeader);
            request = RemotingCommand.createRequestCommand(RequestCode.SEND_REPLY_MESSAGE_V2, requestHeaderV2);
        } else {
            request = RemotingCommand.createRequestCommand(RequestCode.SEND_REPLY_MESSAGE, requestHeader);
        }
    } else {
        if (sendSmartMsg || msg instanceof MessageBatch) {
            SendMessageRequestHeaderV2 requestHeaderV2 = SendMessageRequestHeaderV2.createSendMessageRequestHeaderV2(requestHeader);
            request = RemotingCommand.createRequestCommand(msg instanceof MessageBatch ? RequestCode.SEND_BATCH_MESSAGE : RequestCode.SEND_MESSAGE_V2, requestHeaderV2);
        } else {
            request = RemotingCommand.createRequestCommand(RequestCode.SEND_MESSAGE, requestHeader);
        }
    }
    request.setBody(msg.getBody());

    switch (communicationMode) {
        case ONEWAY:
            this.remotingClient.invokeOneway(addr, request, timeoutMillis);
            return null;
        case ASYNC:
            final AtomicInteger times = new AtomicInteger();
            long costTimeAsync = System.currentTimeMillis() - beginStartTime;
            if (timeoutMillis < costTimeAsync) {
                throw new RemotingTooMuchRequestException("sendMessage call timeout");
            }
            this.sendMessageAsync(addr, brokerName, msg, timeoutMillis - costTimeAsync, request, sendCallback, topicPublishInfo, instance,
                retryTimesWhenSendFailed, times, context, producer);
            return null;
        case SYNC:
            long costTimeSync = System.currentTimeMillis() - beginStartTime;
            if (timeoutMillis < costTimeSync) {
                throw new RemotingTooMuchRequestException("sendMessage call timeout");
            }
            return this.sendMessageSync(addr, brokerName, msg, timeoutMillis - costTimeSync, request);
        default:
            assert false;
            break;
    }

    return null;
}
```

我们应该发现走了这么久,还一直都没到真正的发送阶段,一直在做发送前的准备工作,那接下来应该是了吧,这里我们先选择同步发送的方法点进去看下

```java
private SendResult sendMessageSync(
    final String addr,
    final String brokerName,
    final Message msg,
    final long timeoutMillis,
    final RemotingCommand request
) throws RemotingException, MQBrokerException, InterruptedException {
    /**通过netty发送到broker*/
    RemotingCommand response = this.remotingClient.invokeSync(addr, request, timeoutMillis);
    assert response != null;
    /**将返回结果包装到sendResult*/
    return this.processSendResponse(brokerName, msg, response);
}
```

看来终于要到真正的发送了,这里调用的包装的Netty的通信类的同步数据传输接口,再拿到发送的结果,处理好之后返回给上层.

这下终于要揭开面纱了,进入了网络传输层,原来Netty就是在这里应用的!继续看remotingClient的invokeSync,这个接口是Netty客户端的应用的定义,实现在NettyRemotingClient.

这里是建立通信的channel,然后执行一些钩子函数

```java
public RemotingCommand invokeSync(String addr, final RemotingCommand request, long timeoutMillis)
    throws InterruptedException, RemotingConnectException, RemotingSendRequestException, RemotingTimeoutException {
    long beginStartTime = System.currentTimeMillis();
    /**通过ip地址建立通向地址的channel,如果缓存中有channel就从缓存中取*/
    final Channel channel = this.getAndCreateChannel(addr);
    /**确认channel可用*/
    if (channel != null && channel.isActive()) {
        try {
            /**执行rpc请求前的钩子方法*/
            doBeforeRpcHooks(addr, request);
            long costTime = System.currentTimeMillis() - beginStartTime;
            if (timeoutMillis < costTime) {
                throw new RemotingTimeoutException("invokeSync call timeout");
            }
            RemotingCommand response = this.invokeSyncImpl(channel, request, timeoutMillis - costTime);
            doAfterRpcHooks(RemotingHelper.parseChannelRemoteAddr(channel), request, response);
            return response;
        } catch (RemotingSendRequestException e) {
            log.warn("invokeSync: send request exception, so close the channel[{}]", addr);
            this.closeChannel(addr, channel);
            throw e;
        } catch (RemotingTimeoutException e) {
            if (nettyClientConfig.isClientCloseSocketIfTimeout()) {
                this.closeChannel(addr, channel);
                log.warn("invokeSync: close socket because of timeout, {}ms, {}", timeoutMillis, addr);
            }
            log.warn("invokeSync: wait response timeout exception, the channel[{}]", addr);
            throw e;
        }
    } else {
        this.closeChannel(addr, channel);
        throw new RemotingConnectException(addr);
    }
}
```

进入invokeSyncImpl

这里能看到Netty的数据发送了,因此整个消息的发送就到这里结束了(当然还有将结果一步步回传).可以看到Rocket的底层通信依赖的就是Netty,用Netty实现网络通信也是非常的简单,并且Netty也相当的高效.跑个题:Dubbo的通信也是用的Netty,有时间的话一定要精研一下Netty.

```java
public RemotingCommand invokeSyncImpl(final Channel channel, final RemotingCommand request,
    final long timeoutMillis)
    throws InterruptedException, RemotingSendRequestException, RemotingTimeoutException {
    final int opaque = request.getOpaque();

    try {
        final ResponseFuture responseFuture = new ResponseFuture(channel, opaque, timeoutMillis, null, null);
        this.responseTable.put(opaque, responseFuture);
        final SocketAddress addr = channel.remoteAddress();
        /**原生netty开始传输数据到网络,并建立一个listener监听传输结果*/
        channel.writeAndFlush(request).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture f) throws Exception {
                if (f.isSuccess()) {
                    responseFuture.setSendRequestOK(true);
                    return;
                } else {
                    responseFuture.setSendRequestOK(false);
                }

                responseTable.remove(opaque);
                responseFuture.setCause(f.cause());
                responseFuture.putResponse(null);
                log.warn("send a request command to channel <" + addr + "> failed.");
            }
        });

        RemotingCommand responseCommand = responseFuture.waitResponse(timeoutMillis);
        if (null == responseCommand) {
            if (responseFuture.isSendRequestOK()) {
                throw new RemotingTimeoutException(RemotingHelper.parseSocketAddressAddr(addr), timeoutMillis,
                    responseFuture.getCause());
            } else {
                throw new RemotingSendRequestException(RemotingHelper.parseSocketAddressAddr(addr), responseFuture.getCause());
            }
        }

        return responseCommand;
    } finally {
        this.responseTable.remove(opaque);
    }
}
```

回到三个代码片段之前,我们没讲异步发送,在这里继续看一下异步的.前面讲过同步的了,这里咱们稍微看下注释即可,在这里还有对于rocket的一个容错机制的处理

```java
/**
 * 异步发送与同步发送流程差别不大
 * 主要区别在于异步发送不用返回结果给调用方了,异步发送在方法内处理消息发送结果
 */
private void sendMessageAsync(
    final String addr,
    final String brokerName,
    final Message msg,
    final long timeoutMillis,
    final RemotingCommand request,
    final SendCallback sendCallback,
    final TopicPublishInfo topicPublishInfo,
    final MQClientInstance instance,
    final int retryTimesWhenSendFailed,
    final AtomicInteger times,
    final SendMessageContext context,
    final DefaultMQProducerImpl producer
) throws InterruptedException, RemotingException {
    final long beginStartTime = System.currentTimeMillis();
    this.remotingClient.invokeAsync(addr, request, timeoutMillis, new InvokeCallback() {
        @Override
        public void operationComplete(ResponseFuture responseFuture) {
            long cost = System.currentTimeMillis() - beginStartTime;
            RemotingCommand response = responseFuture.getResponseCommand();
            /**
             * 如果sendCallback不为空的话就不处理sendCallBack了
             */
            if (null == sendCallback && response != null) {

                try {
                    SendResult sendResult = MQClientAPIImpl.this.processSendResponse(brokerName, msg, response);
                    if (context != null && sendResult != null) {
                        context.setSendResult(sendResult);
                        context.getProducer().executeSendMessageHookAfter(context);
                    }
                } catch (Throwable e) {
                }
                /**
                 * mq如果开启了容错策略,rocketmq会通过预测机制来预测一个broker是否可用
                 * 更新当前broker处理一条消息所需要的时间.根据这个时间去预测broker的可用时间
                 *通过currentLatency的大小区间,来预测
                 */
                producer.updateFaultItem(brokerName, System.currentTimeMillis() - responseFuture.getBeginTimestamp(), false);
                return;
            }

            /**
             * 如果sendCallBack为空,那么将结果处理到sendCallBack
             */
            if (response != null) {
                try {
                    SendResult sendResult = MQClientAPIImpl.this.processSendResponse(brokerName, msg, response);
                    assert sendResult != null;
                    if (context != null) {
                        context.setSendResult(sendResult);
                        context.getProducer().executeSendMessageHookAfter(context);
                    }

                    try {
                        sendCallback.onSuccess(sendResult);
                    } catch (Throwable e) {
                    }

                    producer.updateFaultItem(brokerName, System.currentTimeMillis() - responseFuture.getBeginTimestamp(), false);
                } catch (Exception e) {
                    producer.updateFaultItem(brokerName, System.currentTimeMillis() - responseFuture.getBeginTimestamp(), true);
                    onExceptionImpl(brokerName, msg, timeoutMillis - cost, request, sendCallback, topicPublishInfo, instance,
                        retryTimesWhenSendFailed, times, e, context, false, producer);
                }
            } else {
                producer.updateFaultItem(brokerName, System.currentTimeMillis() - responseFuture.getBeginTimestamp(), true);
                if (!responseFuture.isSendRequestOK()) {
                    MQClientException ex = new MQClientException("send request failed", responseFuture.getCause());
                    onExceptionImpl(brokerName, msg, timeoutMillis - cost, request, sendCallback, topicPublishInfo, instance,
                        retryTimesWhenSendFailed, times, ex, context, true, producer);
                } else if (responseFuture.isTimeout()) {
                    MQClientException ex = new MQClientException("wait response timeout " + responseFuture.getTimeoutMillis() + "ms",
                        responseFuture.getCause());
                    onExceptionImpl(brokerName, msg, timeoutMillis - cost, request, sendCallback, topicPublishInfo, instance,
                        retryTimesWhenSendFailed, times, ex, context, true, producer);
                } else {
                    MQClientException ex = new MQClientException("unknow reseaon", responseFuture.getCause());
                    onExceptionImpl(brokerName, msg, timeoutMillis - cost, request, sendCallback, topicPublishInfo, instance,
                        retryTimesWhenSendFailed, times, ex, context, true, producer);
                }
            }
        }
    });
}
```

继续深入走到了RemotingClient的invokeAsync,基本与invokeSync是一样的

```java
public void invokeAsync(String addr, RemotingCommand request, long timeoutMillis, InvokeCallback invokeCallback)
    throws InterruptedException, RemotingConnectException, RemotingTooMuchRequestException, RemotingTimeoutException,
    RemotingSendRequestException {
    long beginStartTime = System.currentTimeMillis();
    final Channel channel = this.getAndCreateChannel(addr);
    if (channel != null && channel.isActive()) {
        try {
            doBeforeRpcHooks(addr, request);
            long costTime = System.currentTimeMillis() - beginStartTime;
            if (timeoutMillis < costTime) {
                throw new RemotingTooMuchRequestException("invokeAsync call timeout");
            }
            this.invokeAsyncImpl(channel, request, timeoutMillis - costTime, invokeCallback);
        } catch (RemotingSendRequestException e) {
            log.warn("invokeAsync: send request exception, so close the channel[{}]", addr);
            this.closeChannel(addr, channel);
            throw e;
        }
    } else {
        this.closeChannel(addr, channel);
        throw new RemotingConnectException(addr);
    }
}
```

再看invokeAsyncImpl,这里与sync的区别主要是对发送失败做了一些额外处理,因为异步的是可以配置重试策略的,其余发送数据就基本一样了.

```java
public void invokeAsyncImpl(final Channel channel, final RemotingCommand request, final long timeoutMillis,
    final InvokeCallback invokeCallback)
    throws InterruptedException, RemotingTooMuchRequestException, RemotingTimeoutException, RemotingSendRequestException {
    long beginStartTime = System.currentTimeMillis();
    //重试次数
    final int opaque = request.getOpaque();
    boolean acquired = this.semaphoreAsync.tryAcquire(timeoutMillis, TimeUnit.MILLISECONDS);
    if (acquired) {
        final SemaphoreReleaseOnlyOnce once = new SemaphoreReleaseOnlyOnce(this.semaphoreAsync);
        long costTime = System.currentTimeMillis() - beginStartTime;
        if (timeoutMillis < costTime) {
            once.release();
            throw new RemotingTimeoutException("invokeAsyncImpl call timeout");
        }

        final ResponseFuture responseFuture = new ResponseFuture(channel, opaque, timeoutMillis - costTime, invokeCallback, once);
        this.responseTable.put(opaque, responseFuture);
        try {
            channel.writeAndFlush(request).addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture f) throws Exception {
                    //如果响应成功,直接返回
                    if (f.isSuccess()) {
                        responseFuture.setSendRequestOK(true);
                        return;
                    }
                    //失败了则重试次数+1
                    requestFail(opaque);
                    log.warn("send a request command to channel <{}> failed.", RemotingHelper.parseChannelRemoteAddr(channel));
                }
            });
        } catch (Exception e) {
            responseFuture.release();
            log.warn("send a request command to channel <" + RemotingHelper.parseChannelRemoteAddr(channel) + "> Exception", e);
            throw new RemotingSendRequestException(RemotingHelper.parseChannelRemoteAddr(channel), e);
        }
    } else {
        if (timeoutMillis <= 0) {
            throw new RemotingTooMuchRequestException("invokeAsyncImpl invoke too fast");
        } else {
            String info =
                String.format("invokeAsyncImpl tryAcquire semaphore timeout, %dms, waiting thread nums: %d semaphoreAsyncValue: %d",
                    timeoutMillis,
                    this.semaphoreAsync.getQueueLength(),
                    this.semaphoreAsync.availablePermits()
                );
            log.warn(info);
            throw new RemotingTimeoutException(info);
        }
    }
}
```

总结:Rocket真正发送消息的过程主要分为2步,一是发送前的准备,包括各种钩子函数,消息校验,按不同的发送策略对消息进行不同的处理,二就是网络发送了,网络通信是依赖Netty完成的,实现非常的简单.

值得重点关注的细节点有很多,比如异步消息情况下对消息压缩的特殊处理,以及重试消息的策略处理.ps:异步消息情况下的消息压缩是因为出过bug,才有了如今的额外处理,之前是没有的.

本文到此结束,感谢阅读!