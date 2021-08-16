RocketMq进阶源码学习之DefaultLitePullConsumer主动拉取消息的源码级流程分析

PullTaskImpl-->Runnable   会提交给线程池去循环执行,初始不停歇无限循环

如果触发流控,修改间隔时间为50ms

流控有4种情况:

1.所有request*10=所有消息数过多

2.单个mq的拉取数过多

3.单个mq的消息大小过大:msgCount*单条消息的body

4.单个mq中前后offset差距过大,即未消费的消息太多

拉取后保存到request

更新下一次要拉取消息的起点offset



pull模式拉取消息也是调用PullApiWrapper.pullKernelImpl,只不过传过去的模式是Sync,没有传回调对象

即消息发过去同步等待拉取的消息结果,需要的时候就pull,掌握主动权

而在pushConsumer中push拉取消息的方法,一直等待broker主动推送消息,其实也是消费者自己去拉取,调用的也是pullKernelImpl,不过传过去的模式是Async,并带有回调对象,即pushConsumer拉取消息是异步的,
结果通过传过去的PullCallBack对象回调到consumer.



pull:掌握主动权,需要的时候调用,主动拉取时会出创建pullTask到consumeQueue里,然后有线程会一直消费队列里的任务

push:会一直被动接受broker的消息推送,如果推送速率过快,而消费速度跟不上的话,会导致消息大量堆积,当堆积超过一定大小之后,pushConsumer会自动进行流控



在ConsumeRequest的Run方法,即消费逻辑中
刚进来时会判断processQueue的dropped状态,但是在消费完之后还会进行第二次判断
因此即使消息消费过来,但如果此时加入了新的消费者或原先的消费者宕机,触发rebalance重新负载
都会导致消息重复消费


