RocketMq延迟消息



首先看Broker存储消息的方法,CommitLog#putMessage

```java
........
// Delay Delivery
//判断是延迟消息
if (msg.getDelayTimeLevel() > 0) {
    if (msg.getDelayTimeLevel() > this.defaultMessageStore.getScheduleMessageService().getMaxDelayLevel()) {
//如果设置的级别超过了最大级别则重置延迟级别到最大级        		  
msg.setDelayTimeLevel(this.defaultMessageStore.getScheduleMessageService().getMaxDelayLevel());
}

    //修改topic为内部主题SCHEDULE_TOPIC_XXX
    topic = TopicValidator.RMQ_SYS_SCHEDULE_TOPIC;
    queueId = ScheduleMessageService.delayLevel2QueueId(msg.getDelayTimeLevel());

    // 记录原topic, queueId
    MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_TOPIC, msg.getTopic());
    MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_QUEUE_ID, String.valueOf(msg.getQueueId()));
    msg.setPropertiesString(MessageDecoder.messageProperties2String(msg.getProperties()));

    //更新topic和queueId
    msg.setTopic(topic);
    msg.setQueueId(queueId);
}
```



延迟消息处理服务ScheduleMessageService

主要作用是消费SCHEDULE_TOPIC_XXX中的消息,再将其投递到目标Topic

它在启动时会创建一个定时器Timer,然后根据延迟级别创建一一对应的task

主要代码在ScheduleMessageService#start

```java
public void start() {
    if (started.compareAndSet(false, true)) {
        //创建定时器timer
        this.timer = new Timer("ScheduleMessageTimerThread", true);
        //针对每一个延迟级别创建一个TimerTask
        //迭代每个延迟级别,delayLevelTable记录了每个延迟级别对应的延迟时间
        for (Map.Entry<Integer, Long> entry : this.delayLevelTable.entrySet()) {
            //获取延迟级别对应的时间
            Integer level = entry.getKey();
            Long timeDelay = entry.getValue();
            Long offset = this.offsetTable.get(level);
            if (null == offset) {
                offset = 0L;
            }
            //创建timerTask
            if (timeDelay != null) {
                this.timer.schedule(new DeliverDelayedMessageTimerTask(level, offset), FIRST_DELAY_TIME);
            }
        }

        this.timer.scheduleAtFixedRate(new TimerTask() {

            @Override
            public void run() {
                try {
                    if (started.get()) ScheduleMessageService.this.persist();
                } catch (Throwable e) {
                    log.error("scheduleAtFixedRate flush exception", e);
                }
            }
        }, 10000, this.defaultMessageStore.getMessageStoreConfig().getFlushDelayOffsetInterval());
    }
}
```

每个TimerTask在检查消息是否到期时,先检查队列的第一天,如果第一条已到期就开始投递到目标topic并检查后面的消息,如果第一条都没有到期就不继续检查后续消息



实际上消息重试也是通过延迟消息来完成的,消息重试最多16次,观察时间间隔可得知这16次就是去掉了延迟级别的前2级后的数据,因此这也是最多重试16次的一个原因

