没有专门发顺序消息的api,不保证Topic层面的消息发送顺序.

但可以多partion发消息时指定partion,或者单partion发消息



rocket支持全局和分mq级的顺序消息,有特定的api



没有api级别的消息重试和死信队列

重试只能通过代码手动重试





事务消息:

rocket保证的是业务上的事务

kafka保证的是多条消息发送的事务,类似于sql中多条sql同时执行



延时消息:

rocket支持

kafka本身不支持,可以开发一个代理服务,用来处理延时和转发消息



消息过滤:

kafka不支持



消息0丢失:

kafka支持,设置每条消息强制刷盘即可



kafka适合业务比较简单的场景,由于不支持多种消息特性,

比如消息过滤,可以用多topic来处理过滤,但是这样比较浪费topic的资源,每个topic的消息量可能差距很大,并且通过细分Topic来进行过滤的话,没法保证业务上的消息顺序

消息重试:可以自己实现多个队列,处理不同的延时级别,每一次重试失败,延时级别提升一级

死信队列:自己实现,重试次数max的消息转向死信队列

延时消息:也可以多队列,多consumer处理不同级别的延时消息

以上处理方式都需要创建多topic,消耗topic的资源,而多Topic达到一定数量之后会很影响kafka(根据磁盘性能变动,一般成百上千之后就会影响)



相对于rocket的优点:

普通情况下吞吐量更高

(发送消息有缓冲池,会多条消息打包成一个batch再压缩发送,当然这样突然宕机的话丢失的消息更多)

更节约空间,消息都是压缩的,当然,压缩会带来性能的消耗



备份机制:

rocket是直接区分broker的身份,分为主从

kafka是通过多partion replication备份,区分partion的身份,一个partion是leader,其余的是follower




