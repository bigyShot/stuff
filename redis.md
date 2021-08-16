redis



主从备份

slave启动的时候,会先ping master,如果是第一次连接,就触发全量复制,master会生成一份rdb文件,传输给slave,这个复制的最大时间默认为60s,如果超过这个时间,就认为slave复制失败,在redis的内存很大的情况下,可以适当调大这个值

增量同步,第一次连接过后,后面再同步就是异步复制写命令.slave不执行key的过期和淘汰,当master淘汰key以后,会模拟一跳del key的命令发送到slave

master每隔10s发heartbeat给slave,slave没1s发一次

断点续传

2.8开始,支持主从复制的断点续传,master 会在内存中维护一个back log,master 和slave都会保存一个 replica offset 和一个 master run id,offset就是保存在back log中,如果master和slave网络连接断掉了,slave就会让master从上次replica offset开始复制,如果没有对应的offset,会执行一次full resynchronization





哨兵机制

哨兵集群最好最少部署3个以上,当quorum数量的哨兵确定master宕机以后,就会开始触发新的master选举

选举策略为先判断slave priority,再看replica offset,都一样再判断run id,优先选择更小的那个slave



自动发现机制

哨兵之间的互相发现是通过rdis的pub/sub机制实现的,每个哨兵都会往_sentinel_:hello这个channel发送一个消息,这时其他所有哨兵都可以感知到这条消息.消息内容是哨兵的host+ip+run id,还有对这个master的监控配置