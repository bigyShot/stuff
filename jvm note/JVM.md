Par New+CMS
进行ygc的时候,如果eden区中还有1个2m的对象和一个500k的对象存活,而在回收时发现survivor只有1m的空间,那么500k的对象会进入survivor
那个2m的对象会直接进入老年代

要优化jvm gc,重点就在于尽量不要让对象进入老年代
1.评估并发量,平均每个任务处理所需的内存大小,评估需要多少台机器,每台机器需要多大的内存
2.根据系统的任务处理速度,评估内存使用情况,然后合理分配eden,survivor,老年代的大小.

总体原则:让对象在YGC时就被回收掉,不要进入老年代,长期存活的对象使其直接进入老年代,避免复制来复制去浪费处理时间.
对系统响应敏感且内存需求大的,建议采用G1.

根据内存增速来评估多久进行以此YGC,
根据每次YGC的存活,评估一下Survivor的大小设置是否合理.
评估多久进行以此FULL GC,STW时间是否可以接受

随着流量增加,负载增加10倍,100倍
1.同比增加机器数量,机器配置不变,可以保持jvm配置不变
2.使用更高的机器配置,如果对响应时间敏感,内存配置提高很多的话,可以考虑使用G1

jvm参数配置应当加入-XX:DisableExplicitGC,禁止显示GC,即禁止在代码中调用System.gc();

在cms的重新标记前触发一次YGC,减少年轻代的对象,有助于提升CMS重新标记阶段的性能.因为年轻代可能有对象和老年代有引用关系,扫描的时候可能会扫到年轻代去

触发FullGc的情景:
1.每次YGC后,存活的对象很多,快速填满old区
2.大对象很多,都直接进入old
3.metaspace被快速填满--加载类太多
4.类存泄露
5.调用System.gc()

-XX:NewSize=5242880 -XX:MaxNewSize=5242880 -XX:InitialHeapSize=10485760 -XX:MaxHeapSize=10485760 -XX:SurvivorRatio=8 
-XX:PretenureSizeThreshold=10485760 -XX:+UseParNewGC -XX:+UseConcMarkSweepGC -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -Xloggc:gc.log

-XX:NewSize=10485760 -XX:MaxNewSize=10485760 -XX:InitialHeapSize=20971520 -XX:MaxHeapSize=20971520 -XX:SurvivorRatio=8
 -XX:MaxTenuringThreshold=15 -XX:PretenureSizeThreshold=10485760 -XX:+UseParNewGC -XX:+UseConcMarkSweepGC -XX:+PrintGCDetails 
 -XX:+PrintGCTimeStamps -Xloggc:gc.log



G1  
因为对于GC的时间有预估,并且分Region,因此jvm要消耗更多的资源在这一部分的计算上,因此更适合cpu计算能力强的
且G1可以设置能接受的GC时间,因此G1非常适合对响应时间要求高的应用,即使GC可能更频繁,但不会明显影响到单批用户的使用体验
对于大堆,内存配置很高的机器,也更适合用G1,因为大内存的机器,比如几十G,那么GC需要的时间会非常多,对于高响应要求的应用是无法接受的

同样,因为G1需要对于Region的数据的计算,对于cpu性能较差的机器来说,par new+cms更适合

G1中大对象不会直接进入老年代,有专门的的大对象Region

新生代占比超过60%的时候,会触发YGC,采用复制算法,将一个region的存活对象挪到另一个region,然后对这个region进行回收

老年代的region占比超过45%的时候,就会触发mixedGc,mixedGc会回收年轻代,老年代以及永久代,
mixedGc过程的最后一步执行混合回收的时候,会停止所有程序运行,也就是stw,
因此G1允许执行多次混合回收,默认值是8次,由-XX:G1MixedGCCountTarget控制
比如预计需要回收160个region,那么就每次回收掉20个region,反复执行8次以达到回收目标
在回收中如果空闲Region大小达到堆的5%,会提前结束回收

在执行mixedGc的时候,因为无论哪个代采用的都是复制算法,万一在拷贝的过程中发现没有空闲的region可以承载存活的对象了,就会触发一次失败.
一旦失败,就会立马切换为停止系统程序,然后采用单线程标记(serial old),清理和压缩.空闲出一批region,这个过程极其缓慢.

G1优化YGC,主要在于设置-XX:MaxGCPauseMills,如果设置过大,可能会导致一次回收Region过多,导致停顿时间太久
如果设置过小,会导致YGC太频繁,因此需要在不停地试探中试探出一个当前业务能够接受的范围.(频次以及停顿时间)

优化MixedGc,还是-XX:MaxGcPauseMills,如果设置过高,导致系统运行很久,新生代可能都占用了堆内存的60%了,此时才出发YGC的话,
那么存活下来的对象就可能会很多,此时就会导致survivor放不下存活的对象,导致直接进入老年代中,
或者存活对象过多,导致进入survivor以后触发了动态年龄判定规则,达到survivor区域的50%,也会导致一些对象进入老年代中

一般的优化策略就是合理加大survivor区的大小,使eden区回收后存活的对象不那么容易进入老年代,就能减少老年代的回收次数	



如果region的存活对象超过85%,就不进行回收,因为回收是采用复制算法,存活对象过多,复制对象太耗费性能.





类加载:

除了main方法所在的类外,其他的都只会在有需要被用到的时候才会加载,而不是一次加载所有

1.验证: 验证.class文件是否符合class文件的内容规范,然后加载到内存中,验证是否符合JVM的规范

2.准备: 给类和类变量分配内存空间,分配默认值.

3.解析: 将符号引用替换为直接引用

初始化:  初始化类变量的值等

加载一个类时会先加载它的父类

双亲委派模型可以避免多层级的加载器结构重复加载某些类.





JVM内存区域分布

方法区(元空间)即MetaSpce: 存放类的各种相关信息

堆

虚拟方法栈

本地方法栈

PC

