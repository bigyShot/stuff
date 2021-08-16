SpringBoot接入最新版RocketMq-Spring2.2.0,消费者指定返回消息处理状态

因为用的是RocketMq4.8.0,因此接入最新的rocketmq-spring

首先引入依赖

```
<dependency>
    <groupId>org.apache.rocketmq</groupId>
    <artifactId>rocketmq-spring-boot-starter</artifactId>
    <version>2.2.0</version>
</dependency>
```

RocketMq-Spring提供了默认的rocket生产者,RocketMqTemplate

直接注入就可以直接使用,默认的是读取application.yml/properties文件里的rocketMq默认配置路径
```java
@Autowired
private RocketMqTemplate rocketMqTemplate
```
默认配置属性如下
```
rocketmq:
  name-server: localhost:9876
  producer:
    group: audit-group
```

也可以自定义RocketMqTemplate,这样就可以同时拥有多个不同配置的生产者,自定义生产者非常简单,只需要直接继承RocketMqTemlate就可以了,然后从注解中配置属性:

```java
@ExtRocketMQTemplateConfiguration(group = "audit-test",nameServer = "localhost:9876")
public class MyRocketMqTemplate extends RocketMQTemplate {
}
```

上面那个注解中可以配置很多属性,可以直接赋值读取,也可以用表达式比如${myrocket.nameserver}从yml/properties配置文件中读取,要用的时候直接注入这个类的对象就可以使用了

```java
@Service
public class Producer {

    String topic="Topic-test";
    //如果需要标签过滤的话,topic可以是Topic:Tag的格式
    String topicWithTag="Topic-test:TagA";
    String msg="hi there!";

    @Autowired
    private MyRocketMqTemplate rocketMQTemplate;

    //同步发送消息
    public void sendSyncMsg(){
        rocketMQTemplate.syncSend(topic,msg);
    }
    
    //异步发送消息
    public void sendAsyncMsg(){
        rocketMQTemplate.asyncSend(topic, msg, new SendCallback() {
            @Override
            public void onSuccess(SendResult sendResult) {
                if(sendResult.getSendStatus()== SendStatus.SEND_OK){
                    System.out.println("send successful");
                }
            }

            @Override
            public void onException(Throwable throwable) {
                System.out.println("send failed");
            }
        });
    }
    
    //发送顺序消息,最好对应的topic的写queue和读queue都设置为1
    public void sendMsgOrderly(){
        String hashKey="orderId:xxxx";
        //hashKey的作用是指定queue,万一topic存在多个queue,可以指定顺序消息生产在这个特点的queue上,比如用orderId指定
        rocketMQTemplate.syncSendOrderly(topic,msg,hashKey);
    }
}
```

再看看消费者的示例:

主要是@RocketMessageListener注解,在其中配置消费者的属性,consumeMode可以配置是顺序消费还是普通消费

```java
@Service
@RocketMQMessageListener(topic = "audit-test",consumerGroup = "broker-a",consumeMode = ConsumeMode.ORDERLY)
public class Consumer implements RocketMQListener<MessageExt>, RocketMQPushConsumerLifecycleListener {
    @Override
    public void onMessage(MessageExt messageExt) {
        System.out.println(messageExt.toString());
    }

    @Override
    public void prepareStart(DefaultMQPushConsumer defaultMQPushConsumer) {
        defaultMQPushConsumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_LAST_OFFSET);
        defaultMQPushConsumer.setConsumeTimestamp(UtilAll.timeMillisToHumanString3(System.currentTimeMillis()));
    }
}
```

细心的读者应该能看到,这里的onMessage方法是void类型的,没有返回状态,与我们平时用的不一样,那如果消费失败,怎么返回RECONSUME_LATER的状态呢,github上官方是回复throw exception的时候会自动处理消息返回RECONSUME_LATER,直接抛出RuntimeException即可.

