关于收到broker返回状态 PCBUSY_CLEAN_QUEUE 状态的思考



收到这种类似的状态返回,一般是broker负载过高,因此直接拒绝当前的发送请求.

BrokerController在启动的时候会启动一个BrokerFastFailure线程池,用来处理broker的快速失败,每隔10ms会执行一次,判断当前的osPageCache是否处于繁忙状态,如果是的话,就会判定当前broker负载过高,会取出当前SendThreadPoolQueue中所有的线程,全部返回PCBUSY_CLEAN_QUEUE的状态.

当然,生产端收到这个返回值后,不会直接抛弃消息,而是会看当前Topic是否在其他Broker上还有分片,如果有的话就会将此条消息重试发送到其他Broker上去.



那这种Broker负载过高的情况,可以考虑为其增加messageQueue,分配到其他的broker上去,减少Broker的压力.

从控制台上可以增加Topic的mq,但这个只能增加到当前所在的Topic上,因此需要增加mq到其他的broker上去的话,需要走mqadmin,执行命令扩容.



![image-20210722181301162](C:\Users\EDZ\Desktop\note'\rocket\img\image-20210722181301162.png)

可以通过 -c 命令来指定在集群中所有的 broker 上创建队列，在本例中，将队列数从 4 设置为 8:

sh ./mqadmin upateTopic -n 192.168.0.220:9876 -c DefaultCluster -t topic_dw_test_by_order_01 -r 8 -w 8 

由mqAdmin收到命令后,先本地持久化一份topic的配置信息.然后再通过netty转发topic的配置信息到namesrv

最终由DefaultRequestProcessor处理这个请求,processor调用RouteInfoManager来管理更新Topic的配置信息

