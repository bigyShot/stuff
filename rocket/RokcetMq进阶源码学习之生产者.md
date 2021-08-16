RocketMq进阶源码学习之生产者启动流程分析

学习源码最好的方式是从应用中学习,从实际应用的某个点入手,一点点追踪底层,然后理解整个流程的原理.而不是这里打开一个类看下,那里打开一个类看下,关联性不强的话,这样看效率是非常低的(个人理解).

这里找个example,单纯简单的发送一条消息,从生产者的start方法开始入手

```java
public static void main(String[] args) throws MQClientException, InterruptedException {
    DefaultMQProducer producer = new DefaultMQProducer("ProducerGroupName");
    producer.start();
    Message msg = new Message("TopicTest",
                        "TagA",
                        "OrderID188",
                        "Hello world".getBytes(RemotingHelper.DEFAULT_CHARSET));
    producer.send()
    producer.shutdown();
}
```

这里是start方法,在里面的start方法下,有几行代码时表示是否开启消息轨迹追踪的,这里对traceDispatcher进行了null判断,这里不明白traceDispatcher是在哪定义的,就一路追踪了下,发现traceDispatcher这个对象是在Producer的构造函数的中进行初始化的,DefaultMqProducer有一个构造函数里有一个参数是enableMsgTrace,如果传入为true,就会初始化traceDispatcher对象,那么在start方法这里判断不为空就会开启消息轨迹追踪了

比如这样的构造函数即可开启消息轨迹追踪

```java
DefaultMQProducer producer = new DefaultMQProducer("ProducerGroupName",true);
```



```java
public void start() throws MQClientException {
    //设置生产者组名
    this.setProducerGroup(withNamespace(this.producerGroup));
    this.defaultMQProducerImpl.start();
    //traceDispatcher是为初始化,已初始化则表示开启消息追踪
    if (null != traceDispatcher) {
        try {
            traceDispatcher.start(this.getNamesrvAddr(), this.getAccessChannel());
        } catch (MQClientException e) {
            log.warn("trace dispatcher start failed ", e);
        }
    }
}
```

然后再经过一些配置的校验之后,开始启动,主要是做一些定时任务与支线服务线程(如消息重平衡服务,拉取服务等等)

```java
public void start() throws MQClientException {
    synchronized (this) {
        switch (this.serviceState) {
            case CREATE_JUST:
                this.serviceState = ServiceState.START_FAILED;
                // If not specified,looking address from name server
                if (null == this.clientConfig.getNamesrvAddr()) {
                    this.mQClientAPIImpl.fetchNameServerAddr();
                }
                //开启netty客户端,NettyRemotingClient
                this.mQClientAPIImpl.start();
                //开启多个定时任务线程池,如发送心跳,持久化消息消费的offset等
                this.startScheduledTask();
                //开启拉取消息服务
                this.pullMessageService.start();
                //开启重均衡消息服务
                this.rebalanceService.start();
                // Start push service
                this.defaultMQProducer.getDefaultMQProducerImpl().start(false);
                log.info("the client factory [{}] start OK", this.clientId);
                this.serviceState = ServiceState.RUNNING;
                break;
            case START_FAILED:
                throw new MQClientException("The Factory object[" + this.getClientId() + "] has been created before, and failed.", null);
            default:
                break;
        }
    }
```

生产者启动基本就到此为止了,就是做一些校验,看看是否需要开启消息轨迹追踪,再启动Netty客户端,然后在启动一些辅助服务就启动完毕了