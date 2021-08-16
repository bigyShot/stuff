1.Gitee通过http发消息到Proxy,Proxy-->MQ,Review从MQ消费.

​	1.如果是异步回调接口,Review直接处理完后回调

​	2.review处理完后,将结果发送到MQ,然后自己慢慢消费

2.Gitee直接调Review

​	1.Review直接处理

​	2.Review塞入MQ,异步处理



处理完后的回调结果塞MQ已经没什么意义了,直接回调也只是一次简单的http消费(可能并发特别高的时候回调失败,在mq中的话可以确保回调成功)

请求进来后塞MQ,可以一定程度上削峰,审查的逻辑处理肯定比直接塞MQ要复杂的多,时间要长的多

有2种Topic,1个是giteeRequest,1个是giteeCallback



做成Http的代理后,Gitee加一个线程池,定时去拉取消息.





定义好Http Proxy的接口格式

```json
{
	"topic":"review",  //主题
    "accessKey":"", //账户
    "secretKey":"",	//密码
    "data":""  //参数,json格式
}
```

公钥秘钥是客户端面向broker的

​	

如果是集群消费消息怎么办,做一个普通的拉取消息接口肯定不行

没法做到负载均衡,

消息存到Proxy:   

接口取走的消息直接从队列中踢掉:  万一消费失败了呢?没有持久化?消费超时了没有返回ack呢



RocketMq部署文档

RocketMq架构图  概念分层拓扑图

评估 审核平台所需要的机器,RocketMq在多少流量情况下需要多大的机器

20kb x 10x 100=20mb/s 

4g  2+2  2=1.6+4  1ygc/80s

xms=16g xmx=16g xmn=12g 

9.6:1.2:1.2

1ygc/8min  

100qps会生成大约20m的对象,那1s能不能处理完100q,如果处理不完就会一直往后面堆,那么ygc时就会有这480s累积下来的没处理完的对象存活进入survior,如果这部分超过0.6g就麻烦了,下次就会送入old.

如果峰值100qps,那么真正1s能处理的数算90  

1/10剩下,960M, 

9.6+0.96=10.56/10--->1.05



8G for jvm

6~7 for heap

4g for eden 

3.6 for eden 2*0.2 for survivor

3600/20=180s 3min



-XX:+CMSParalleInitialMarkEnabled   使fgc的初始标记阶段多线程并发执行
-XX:+CMSScavengeBeforeRemark   在进行fgc执行,尽量先执行一次ygc(先回收掉一些eden的对象,那CMS在重新标记阶段就可以少扫描一些对象,提升性能,减少了耗时)







要让ygc后存活的对象不超过survior的50%,不然就会触发>50%的同龄对象晋升的原则





记录每一次审查的策略参数以及处理结果

处理一下三方审核平台的结果,制定状态码

封禁用户的接口,记录谁封禁的这个用户

发过来的请求里带有用户的信息,知道这个用户被封了多少信息了

来源加上用户ip信息

根据审查结果的等级进行计算,制定一个封禁策略

展示被封禁信息用户排名,比如小明封了500条

检测是否一样的数据,如果存在一样的可以不用审查,直接拿结果



长文本分段审查,直接多线程处理还是一段一段查,第一段就违规了那后面的子段还需不需要继续查

幂等性检测换个数据源 redis->es



分段切割-->把多段文本的处理结果汇总-->存至es



分段审核多段的结果可能是不一样的,必须拿到所有的response再统一处理

设计好枚举类以后还得把它替换掉现有的业务



blacklist-->es-->review



优化黑名单校验ac自动机



组一个list,放大概百来个违规词,然后找一本普通的书,随机截取长度,然后再随机插入违规词,实现测试案例制造成功



请求参数 20k

发消息20k

收消息20k



nohup java -jar -XX:NewSize=3G -XX:MaxNewSize=3G -XX:InitialHeapSize=6G -XX:MaxHeapSize=6G -XX:SurvivorRatio=8 -XX:+DisableExplicitGC -XX:PretenureSizeThreshold=2M -XX:+UseParNewGC -XX:+UseConcMarkSweepGC -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -Xloggc:gc.log -XX:MetaspaceSize=300M -XX:MetaspaceSize=300M admin.jar >log.log &

在100qps持续60s的情况下,cpu占用率最高到25%,一般在15	作用,survivor区每次存活100M左右,会有20M对象进入old

60s内共产生3328M对象,发生了1次ygc,平均每秒55m,即100qps下平均产生对象55m/s



1122   7ygc

2362   19ygc

12*2500+1200=31200   31200/300=100m

200qps持续300s的情况下,平均每s生成104m对象,发生了12次ygc,300/12=25,平均ygc/25s,cpu占用率一般在25作用

old区达到了630m,增长了94m,平均每次ygc增长old区13m,平均之前的按old区每次ygc增长15m来看,预估每83min分钟要发生一次fgc,

每个survivor区的大小是314M,在有些ygc过程中,s区的占用比例接近了50%,初步判断是因为大于50%的原因导致进入了一部分到old,

但是s区接近50%的次数是很少的,因此增长的94m的old对象里,很大概率是因为年龄达到了15晋升的,或者是大于2M的大对象进入

复盘思路:

由于定时了每5min,会从数据库拉一次全量keyword构建keywordTrie,如果这个数足够大,那么查出来的List和构建好的Tire都会成为大对象,因此每5min会刷

2-3个大对象进入old



优化思路:

由于是审核结果的处理,如果提高大对象的门槛,那么如果keywordTrie达不到门槛大小,那将会在ygc中反复复制移动,浪费操作效率.

但是keywordTrie最多5min就会更新一次,那只要控制好5min内的ygc次数就好



1652条记录下,产生对象18.4M





异常有几种

1.errorCode

2.超时

3.账号空







nohup ./mysqld_exporter --collect.info_schema.processlist --collect.info_schema.innodb_tablespaces --collect.info_schema.innodb_metrics --collect.perf_schema.tableiowaits --collect.perf_schema.indexiowaits --collect.perf_schema.tablelocks --collect.engine_innodb_status --collect.perf_schema.file_events --collect.binlog_size --collect.info_schema.clientstats --collect.perf_schema.eventswaits &





账号一直在缓存里,触发了新增或更改账号的方法时刷新一下账号的缓存

然后定时刷新账号的数据到mysql

count_day的数据就不用管,定时刷就ok

