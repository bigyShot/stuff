**RocketMQ部署细节**

RocketMQ

版本:4.8.0

jdk要求:1.8

地址:https://github.com/apache/rocketmq

为保证数据安全与机器负载,选用分布式集群部署模式部署Broker,Namesrv可以采用集群模式确保万无一失

这种模式又有两种模式:

1.多主多从

正常的配备多master,每个master可以配备多slave,但是master挂了的话,slave无法自动切换到master,会导致master所包含的Topic分片不可写,仍可读.

2.Dledger集群

Rocket4.5.0以后新增的功能,一个Dledger集群至少需要三台机器,一主两从,在master挂了的时候,slave可以自动切换为master,继续提供写入的功能,采用raft协议选举.

 

安装过程:

1.下载源码

2.解压   unzip  rocketmq-all-4.8.0-source-release.zip

3.编译 进入rocketmq主目录,使用maven编译 mvn -Prelease-all -DskipTests clean install -U 

4.切换到 distribution/target目录,找到编译完成后的RocketMq

5.配置配置文件

6.启动namesrv ./mqnamesrv

7.启动broker ./mqbroker



安装可视化工具RocketMQ-Console

1.下载 https://github.com/apache/rocketmq-externals/releases/tag/rocketmq-console-1.0

2.maven编译  mvn clean package -Dmaven.test.skip=true

3.java -jar rocketmq-console-ng-1.0.1.jar --rocketmq.config.namesrvAddr=127.0.0.1:9876

namesrvAddr指定的是namesrv的地址



