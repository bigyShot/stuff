RocketMq源码分析之DefaultLitePullConsumer主动拉取消息分析consumeRequestQueue

因为做RMQ的proxy的消费者的时候,消息消费只能主动拉取,然后就想去研究下RMQ中主动拉消息的消费者的源码,开始在网上搜到的都是讲DefaultMQPullConsumer的,然后我用的是RocketMq4.8.0的版本,在4.8中,这个类被标记为将要废弃,将被DefaultLitePullConsumer替代,于是今天就咱们就研究下DefaultLitePullConsumer.

和普通push模式的消费者一样是创建好对象,然后start,它的区别是需要主动去拿消息,然后去消费.

这里面又有两种拉取消息的方式,一种是assign,一种是subscribe.

**assign是指定哪些队列去拉取,subscribe是只指定topic,然后由均衡策略去所有队列中选择队列拉取**

这是subscribe

```java
        DefaultLitePullConsumer litePullConsumer = new DefaultLitePullConsumer("lite_pull_consumer_test");
        litePullConsumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);
        litePullConsumer.subscribe("TopicTest", "*");
        litePullConsumer.start();
        try {
            while (running) {
                List<MessageExt> messageExts = litePullConsumer.poll();
                System.out.printf("%s%n", messageExts);

            }
        } finally {
            litePullConsumer.shutdown();
        }
```
这是assign
```java
        DefaultLitePullConsumer litePullConsumer = new DefaultLitePullConsumer("please_rename_unique_group_name");
        litePullConsumer.setAutoCommit(false);
        litePullConsumer.start();
        Collection<MessageQueue> mqSet = litePullConsumer.fetchMessageQueues("TopicTest");
        List<MessageQueue> list = new ArrayList<>(mqSet);
        List<MessageQueue> assignList = new ArrayList<>();
        for (int i = 0; i < list.size() / 2; i++) {
            assignList.add(list.get(i));
        }
        litePullConsumer.assign(assignList);
        litePullConsumer.seek(assignList.get(0), 10);
        try {
            while (running) {
                List<MessageExt> messageExts = litePullConsumer.poll();
                System.out.printf("%s %n", messageExts);
                litePullConsumer.commitSync();
            }
        } finally {
            litePullConsumer.shutdown();
        }
```

首先看start方法干了什么
```java
    public synchronized void start() throws MQClientException {
        switch (this.serviceState) {
            case CREATE_JUST:
                this.serviceState = ServiceState.START_FAILED;
                //这里是对consumerGroupName以及messageModel等一些基础属性做一下校验
                this.checkConfig();

                //如果是集群模式,就将实例名改为PID,因为一般没设置instanceName的情况下,默认是Default,
                //但是集群下,不同的consumer节点需要区分开来,不能全设置为Default
                if (this.defaultLitePullConsumer.getMessageModel() == MessageModel.CLUSTERING) {
                    this.defaultLitePullConsumer.changeInstanceNameToPID();
                }

                //初始化客户端实例,注册为消费者
                initMQClientFactory();

                //初始化消息的重平衡策略的配置,
                initRebalanceImpl();

                initPullAPIWrapper();

                //根据消息的消费模式,启动不同的offsetStore对象
                //集群模式就启动集群模式的
                //广播模式启动广播模式的
                initOffsetStore();

                mQClientFactory.start();

                //周期性获取最新的Topic和与之对应的MessageQueue
                startScheduleTask();

                this.serviceState = ServiceState.RUNNING;

                log.info("the consumer [{}] start OK", this.defaultLitePullConsumer.getConsumerGroup());

                //根据拉取消息的策略来执行,有两种,一种是订阅Topic,一种是自己分配messageQueue,也就是assign
                //如果是assign,需要在start consumer之前,自己先根据topic拿到所有的messageQueue的信息
                //然后assign自己想要选择的messageQueue
                //如果是subscribe,就会自动根据传入的topic去namesrv拉取所有messageQueue的数据
                operateAfterRunning();

                break;
            case RUNNING:
            case START_FAILED:
            case SHUTDOWN_ALREADY:
                throw new MQClientException("The PullConsumer service state not OK, maybe started once, "
                    + this.serviceState
                    + FAQUrl.suggestTodo(FAQUrl.CLIENT_SERVICE_NOT_OK),
                    null);
            default:
                break;
        }
    }
```

第一遍看源码的时候,就这么大致过了一遍,然后接下来就是主动拉取消息的时候了:
```java
List<MessageExt> messageExts = litePullConsumer.poll();
```
在最上面启动消费者的代码中有这么一行,是主动拉取消息的方法,在以前的DefaultMQPullConsumer中,是可以设置拉取多少条消息的,但是现在这个类不能设置了,我点进poll方法开始追踪消息的拉取策略
```java
    public synchronized List<MessageExt> poll(long timeout) {
        try {
            checkServiceState();
            if (timeout < 0)
                throw new IllegalArgumentException("Timeout must not be negative");

            //看是否设置了自动提交offset,如果是就根据当前时间判断一下当前是否需要提交一次offset
            if (defaultLitePullConsumer.isAutoCommit()) {
                maybeAutoCommit();
            }
            long endTime = System.currentTimeMillis() + timeout;

            //获取拉取到的消息
            ConsumeRequest consumeRequest = consumeRequestCache.poll(endTime - System.currentTimeMillis(), TimeUnit.MILLISECONDS);

            if (endTime - System.currentTimeMillis() > 0) {
                while (consumeRequest != null && consumeRequest.getProcessQueue().isDropped()) {
                    consumeRequest = consumeRequestCache.poll(endTime - System.currentTimeMillis(), TimeUnit.MILLISECONDS);
                    if (endTime - System.currentTimeMillis() <= 0)
                        break;
                }
            }

            if (consumeRequest != null && !consumeRequest.getProcessQueue().isDropped()) {
                List<MessageExt> messages = consumeRequest.getMessageExts();
                long offset = consumeRequest.getProcessQueue().removeMessage(messages);
                assignedMessageQueue.updateConsumeOffset(consumeRequest.getMessageQueue(), offset);
                //If namespace not null , reset Topic without namespace.
                this.resetTopic(messages);
                return messages;
            }
        } catch (InterruptedException ignore) {

        }

        return Collections.emptyList();
    }
```

看到这里我就发现不对劲了,**ConsumeRequest**是一个保存了消息数据的一个类,这个类居然是从**consumeRequestCache**这么一个BlockingQueue中取出来的
```java
private final BlockingQueue<ConsumeRequest> consumeRequestCache = new LinkedBlockingQueue<ConsumeRequest>();
```
等等,不对劲啊,怎么直接就从程序的队列中取出来了,这啥时候放进去的啊,我都还没开始拉取,怎么就有消息了呢,于是我就追踪了一下这个consumeRequestCache,发现他里面的数据是通过**PullTaskImpl**这个线程类在启动的时候放进去的,然后就发现了一个存放PullTaskImpl的队列
```java
private final ConcurrentMap<MessageQueue, PullTaskImpl> taskTable =
        new ConcurrentHashMap<MessageQueue, PullTaskImpl>();
```
最终追踪到了一个startPullTask方法,在这个方法需要传入一个集合的messageQueue,然后创建一个PullTaskImpl开始拉取消息,生成ConsumeRequest.
继续追踪发现有两个地方使用到了这个startPullTask方法,一个是**updateAssignPullTask**,一个是**updatePullTask**,然后我一路追踪这两个方法的用途.
发现了updateAssignPullTask是在上面start方法中的operateAfterRunning()使用的,这下对应上了,在assign模式下,传入了assign的messageQueues然后在start之后从这些queue拉取消息.

另一个方法是在触发消息分发重平衡的时候执行的.在RebalanceLitePullImpl中

这下主动拉取消息的逻辑明白了,原来是在consumer启动的时候,如果指定了messageQueue,就开始从这些queue中拉取消息,如果没有指定,那就在重平衡的时候从订阅的Topic里拉取所有的messageQueue,然后再拉取消息,之后就通过poll方法来取.

对了还有非常重要的一点,主动拉取消息的情况,是默认自动提交消费位点offset的,可以通过setAutoCommit(false);来设置为手动提交.消费完了记得调用commitSync();