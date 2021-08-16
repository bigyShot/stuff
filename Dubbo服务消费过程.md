Dubbo服务消费过程



ReferenceConfig执行init(),在init中createProxy(),createProxy判断是同一个JVM中调用还是远端调用,初始化方式不同.通过Protocol.refer方法生成invoker对象,invoker是原接口的一个包装对象,最后再通过ProxyFactory.getProxy生成动态代理的对象,@Reference注入的对象本质就是这个动态代理,执行具体方法在代理的InvocationHandler的invoke方法调用



生产者初始化过程

ServiceConfig

Protocol拿到真实的接口ref,经由















布隆过滤器的底层数据结构是一个一维数组,初始化值全部为0,对需要加入进来的变量进行多次hash,每次hash定位到一个index,将hash出来的index位置的值全改为1,这样判断是否存在时候,对变量进行多次hash,如果每一个位置都为1,那么就大概率存在,但是有可能会误判.



如果删除了数据,不能直接更改布隆过滤器的数组的相关位置的值,可以定期重构整个布隆过滤器



数组越大,误判的可能性就越小,多次hash也可以减少误判可能性,但hash次数会直接影响判断的性能