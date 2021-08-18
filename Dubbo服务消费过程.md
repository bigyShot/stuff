Dubbo服务消费过程



ReferenceConfig执行init(),在init中createProxy(),createProxy判断是同一个JVM中调用还是远端调用,初始化方式不同.通过Protocol.refer方法生成invoker对象,invoker是原接口的一个包装对象,最后再通过ProxyFactory.getProxy生成动态代理的对象,@Reference注入的对象本质就是这个动态代理,执行具体方法在代理的InvocationHandler的invoke方法调用



生产者初始化过程

ServiceConfig

Protocol拿到真实的接口ref,经由



dubbo在producer端可以配置

timeout超时时间  retryTimes重试次数  loadBalance负载均衡策略  actives并发度限制



负载均衡策略有4种

random 按权重随机

round robin 轮询

leastActive 最少调用优先,同级随机

consistentHash  一致性hash,同样参数的每次发往同一台节点



dubbo的服务宕机剔除是基于zookeeper的临时节点原理



读服务建议使用failover,失败自动切换,默认重试两次其他服务器

写操作建议使用failfast快速失败,发一次调用失败就立即报错



dubbo会对结果进行自动缓存,用于加速热门数据的访问



dubbo的同步调用是阻塞的



dubbo兼容旧版本服务,通过版本号不同区分,同时上线多版本的服务



当一个接口有多种实现时,可以用group属性来区分,服务方和消费方指定同一个group即可



允许点对点直连,直接测试某一个服务



容错方案设置

Failover失败自动切换其他服务器

Failfast快速失败,立即报错

Failsafe失败安全,忽略出现的异常

Failback失败自动恢复,记录请求,定时重试

Forking并行调用多个服务,只要有一个成功即返回



默认序列化采用Hessian,还有dubbo,fastjson,serialize



dubbo默认使用zookeeper作为注册中心,还有redis,Multicast,simple注册中心,但不推荐使用



dubbo内置了spring,jetty,log4j 服务容器















布隆过滤器的底层数据结构是一个一维数组,初始化值全部为0,对需要加入进来的变量进行多次hash,每次hash定位到一个index,将hash出来的index位置的值全改为1,这样判断是否存在时候,对变量进行多次hash,如果每一个位置都为1,那么就大概率存在,但是有可能会误判.



如果删除了数据,不能直接更改布隆过滤器的数组的相关位置的值,可以定期重构整个布隆过滤器



数组越大,误判的可能性就越小,多次hash也可以减少误判可能性,但hash次数会直接影响判断的性能