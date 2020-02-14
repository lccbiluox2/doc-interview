[TOC]

# flink

##1.简单介绍一下Flink  

Flink核心是一个流式的数据流执行引擎，其针对数据流的分布式计算提供了数据分布、数据通信以及容错机制等功能。基于流执行引擎，Flink提供了诸多更高抽象层的API以便用户编写分布式任务：

DataSet API， 对静态数据进行批处理操作，将静态数据抽象成分布式的数据集，用户可以方便地使用Flink提供的各种操作符对分布式数据集进行处理，支持Java、Scala和Python。

DataStream API，对数据流进行流处理操作，将流式的数据抽象成分布式的数据流，用户可以方便地对分布式数据流进行各种操作，支持Java和Scala。

Table API，对结构化数据进行查询操作，将结构化数据抽象成关系表，并通过类SQL的DSL对关系表进行各种查询操作，支持Java和Scala。

此外，Flink还针对特定的应用领域提供了领域库，例如：

Flink ML，Flink的机器学习库，提供了机器学习Pipelines API并实现了多种机器学习算法。

Gelly，Flink的图计算库，提供了图计算的相关API及多种图计算算法实现。



###  2.Operator Chains

（算子链）这个概念你了解吗？Flink是如何优化的？什么情况下Operator才会chain在一起？

 为了更高效地分布式执行，Flink会尽可能地将operator的subtask链接（chain）在一起形成task。每个task在一个线程中执行。将operators链接成task是非常有效的优化：它能减少线程之间的切换，减少消息的序列化/反序列化，减少数据在缓冲区的交换，减少了延迟的同时提高整体的吞吐量。

两个operator chain在一起的的条件：

1. 上下游的并行度一致

2. 下游节点的入度为1 （也就是说下游节点没有来自其他节点的输入）

3. 上下游节点都在同一个 slot group 中（下面会解释 slot group）
4. 下游节点的 chain 策略为 ALWAYS（可以与上下游链接，map、flatmap、filter等默认是ALWAYS）
5. 上游节点的 chain 策略为 ALWAYS 或 HEAD（只能与下游链接，不能与上游链接，Source默认是HEAD）
6. 两个节点间数据分区方式是 forward（参考理解数据流的分区）
7. 用户没有禁用 chain

 ### 3.**Flink1.9的新特性？**

支持hive读写，支持UDF
Flink SQL TopN和GroupBy等优化
Checkpoint跟savepoint针对实际业务场景做了优化
Flink state查询

### 4.**消费kafka数据的时候，如何处理脏数据？**

可以在处理前加一个fliter算子，将不符合规则的数据过滤出去。

### 5. Flink任务，delay极高，请问你有什么调优策略？

 首先要确定问题产生的原因，找到最耗时的点，确定性能瓶颈点。比如任务频繁反压，找到反压点。主要通过：资源调优、作业参数调优。资源调优即是对作业中的Operator的并发数（parallelism）、CPU（core）、堆内存（heap_memory）等参数进行调优。作业参数调优包括：并行度的设置，State的设置，checkpoint的设置。

### 6.**Flink有没有重启策略？说说有哪几种？**

Flink 实现了多种重启策略。

- 固定延迟重启策略（Fixed Delay Restart Strategy）
- 故障率重启策略（Failure Rate Restart Strategy）
- 没有重启策略（No Restart Strategy）
- Fallback重启策略（Fallback Restart Strategy）

### 7.Task slot是什么

Task slot是一个TaskManager内资源分配的最小载体，代表了一个固定大小的资源子集，每个TaskManager会将其所占有的资源平分给它的slot。 

通过调整 task slot 的数量，用户可以定义task之间是如何相互隔离的。每个 TaskManager 有一个slot，也就意味着每个task运行在独立的 JVM 中。每个 TaskManager 有多个slot的话，也就是说多个task运行在同一个JVM中。 
而在同一个JVM进程中的task，可以共享TCP连接（基于多路复用）和心跳消息，可以减少数据的网络传输，也能共享一些数据结构，一定程度上减少了每个task的消耗





 



Flink面试题，有时间会解决这些问题：

1、为何我使用 ValueState 保存状态 Job 恢复是状态没恢复？

2、flink中watermark究竟是如何生成的，生成的规则是什么，怎么用来处理乱序数据

3、消费kafka数据的时候，如果遇到了脏数据，或者是不符合规则的数据等等怎么处理呢？

4、在Kafka 集群中怎么指定读取/写入数据到指定broker或从指定broker的offset开始消费？

5、Flink能通过oozie或者azkaban提交吗？

6、jobmanager挂掉后，提交的job怎么不经过手动重新提交执行？

7、使用flink-web-ui提交作业并执行 但是/opt/flink/log目录下没有日志文件 请问关于flink的日志（包括jobmanager、taskmanager、每个job自己的日志默认分别存在哪个目录 ）需要怎么配置？

8、通过flink 仪表盘提交的jar 是存储在哪个目录下？

9、从Kafka消费数据进行etl清洗，把结果写入hdfs映射成hive表，压缩格式、hive直接能够读取flink写出的文件、按照文件大小或者时间滚动生成文件

10、flink jar包上传至集群上运行，挂掉后，挂掉期间kafka中未被消费的数据，在重新启动程序后，是自动从checkpoint获取挂掉之前的kafka offset位置，自动消费之前的数据进行处理，还是需要某些手动的操作呢？

11、flink 启动时不自动创建 上传jar的路径，能指定一个创建好的目录吗

12、Flink sink to es 集群上报 slot 不够，单机跑是好的，为什么？

13、Fllink to elasticsearch如何创建索引文档期时间戳？

14、blink有没有api文档或者demo，是否建议blink用于生产环境。

15、flink的Python api怎样？bug多吗？

16、Flink VS Spark Streaming VS Storm VS Kafka Stream

17、你们做实时大屏的技术架构是什么样子的？flume→kafka→flink→redis，然后后端去redis里面捞数据，酱紫可行吗？

18、做一个统计指标的时候，需要在Flink的计算过程中多次读写redis，感觉好怪，星主有没有好的方案？

19、Flink 使用场景大分析，列举了很多的常用场景，可以好好参考一下

20、将kafka中数据sink到mysql时，metadata的数据为空，导入mysql数据不成功？？？

21、使用了ValueState来保存中间状态，在运行时中间状态保存正常，但是在手动停止后，再重新运行，发现中间状态值没有了，之前出现的键值是从0开始计数的，这是为什么？是需要实现CheckpointedFunction吗？

22、flink on yarn jobmanager的HA需要怎么配置。还是说yarn给管理了

23、有两个数据流就行connect，其中一个是实时数据流（kafka 读取)，另一个是配置流。由于配置流是从关系型数据库中读取，速度较慢，导致实时数据流流入数据的时候，配置信息还未发送，这样会导致有些实时数据读取不到配置信息。目前采取的措施是在connect方法后的flatmap的实现的在open 方法中，提前加载一次配置信息，感觉这种实现方式不友好，请问还有其他的实现方式吗？

24、Flink能通过oozie或者azkaban提交吗？

25、不采用yarm部署flink，还有其他的方案吗？ 主要想解决服务器重启后，flink服务怎么自动拉起？ jobmanager挂掉后，提交的job怎么不经过手动重新提交执行？

26、在一个 Job 里将同份数据昨晚清洗操作后，sink 到后端多个地方（看业务需求），如何保持一致性？（一个sink出错，另外的也保证不能插入）

27、flink sql任务在某个特定阶段会发生tm和jm丢失心跳，是不是由于gc时间过长呢，

28、有这样一个需求，统计用户近两周进入产品详情页的来源（1首页大搜索，2产品频道搜索，3其他），为php后端提供数据支持，该信息在端上报事件中，php直接获取有点困难。 我现在的解决方案 通过flink滚动窗口（半小时），统计用户半小时内3个来源pv，然后按照日期序列化，直接写mysql。php从数据库中解析出来，再去统计近两周占比。 问题1，这个需求适合用flink去做吗？ 问题2，我的方案总感觉怪怪的，有没有好的方案？

29、一个task slot 只能同时运行一个任务还是多个任务呢？如果task slot运行的任务比较大，会出现OOM的情况吗？

30、你们怎么对线上flink做监控的，如果整个程序失败了怎么自动重启等等

31、flink cep规则动态解析有接触吗？有没有成型的框架？

32、每一个Window都有一个watermark吗？window是怎么根据watermark进行触发或者销毁的？

33、 CheckPoint与SavePoint的区别是什么？

34、flink可以在算子中共享状态吗？或者大佬你有什么方法可以共享状态的呢？

35、运行几分钟就报了，看taskmager日志，报的是 failed elasticsearch bulk request null，可是我代码里面已经做过空值判断了呀 而且也过滤掉了，flink版本1.7.2 es版本6.3.1

36、这种情况，我们调并行度 还是配置参数好

37、大家都用jdbc写，各种数据库增删查改拼sql有没有觉得很累，ps.set代码一大堆，还要计算每个参数的位置

38、关于datasource的配置，每个taskmanager对应一个datasource?还是每个slot? 实际运行下来，每个slot中datasorce线程池只要设置1就行了，多了也用不到?

39、kafka现在每天出现数据丢失，现在小批量数据，一天200W左右, kafka版本为 1.0.0，集群总共7个节点，TOPIC有十六个分区，单条报文1.5k左右

40、根据key.hash的绝对值 对并发度求模，进行分组，假设10各并发度，实际只有8个分区有处理数据，有2个始终不处理，还有一个分区处理的数据是其他的三倍，如截图

41、flink每7小时不知道在处理什么， CPU 负载 每7小时，有一次高峰，5分钟内平均负载超过0.8，如截图

42、有没有Flink写的项目推荐？我想看到用Flink写的整体项目是怎么组织的，不单单是一个单例子

43、Flink 源码的结构图

44、我想根据不同业务表（case when）进行不同的redis sink（hash ，set），我要如何操作？

45、这个需要清理什么数据呀，我把hdfs里面的已经清理了 启动还是报这个

46、 在流处理系统，在机器发生故障恢复之后，什么情况消息最多会被处理一次？什么情况消息最少会被处理一次呢？

47、我检查点都调到5分钟了，这是什么问题

48、reduce方法后 那个交易时间 怎么不是最新的，是第一次进入的那个时间，

49、Flink on Yarn 模式，用yarn session脚本启动的时候，我在后台没有看到到Jobmanager，TaskManager，ApplicationMaster这几个进程，想请问一下这是什么原因呢？因为之前看官网的时候，说Jobmanager就是一个jvm进程，Taskmanage也是一个JVM进程

50、Flink on Yarn的时候得指定 多少个TaskManager和每个TaskManager slot去运行任务，这样做感觉不太合理，因为用户也不知道需要多少个TaskManager适合，Flink 有动态启动TaskManager的机制吗。

51、参考这个例子，Flink 零基础实战教程：如何计算实时热门商品 | Jark's Blog， 窗口聚合的时候，用keywindow，用的是timeWindowAll，然后在aggregate的时候用aggregate(new CustomAggregateFunction(), new CustomWindowFunction())，打印结果后，发现窗口中一直使用的重复的数据，统计的结果也不变，去掉CustomWindowFunction()就正常了 ？ 非常奇怪

52、用户进入产品预定页面（端埋点上报），并填写了一些信息（端埋点上报），但半小时内并没有产生任何订单，然后给该类用户发送一个push。 1. 这种需求适合用flink去做吗？2. 如果适合，说下大概的思路

53、业务场景是实时获取数据存redis，请问我要如何按天、按周、按月分别存入redis里？（比方说过了一天自动换一个位置存redis）

54、有人 AggregatingState 的例子吗, 感觉官方的例子和 官网的不太一样?

55、flink-jdbc这个jar有吗？怎么没找到啊？1.8.0的没找到，1.6.2的有

56、现有个关于savepoint的问题，操作流程为，取消任务时设置保存点，更新任务，从保存点启动任务；现在遇到个问题，假设我中间某个算子重写，原先通过state编写，有用定时器，现在更改后，采用窗口，反正就是实现方式完全不一样；从保存点启动就会一直报错，重启，原先的保存点不能还原，此时就会有很多数据重复等各种问题，如何才能保证数据不丢失，不重复等，恢复到停止的时候，现在想到的是记下kafka的偏移量，再做处理，貌似也不是很好弄，有什么解决办法吗

57、需要在flink计算app页面访问时长，消费Kafka计算后输出到Kafka。第一条log需要等待第二条log的时间戳计算访问时长。我想问的是，flink是分布式的，那么它能否保证执行的顺序性？后来的数据有没有可能先被执行？

58、我公司想做实时大屏，现有技术是将业务所需指标实时用spark拉到redis里存着，然后再用一条spark streaming流计算简单乘除运算，指标包含了各月份的比较。请问我该如何用flink简化上述流程？

59、flink on yarn 方式，这样理解不知道对不对，yarn-session这个脚本其实就是准备yarn环境的，执行run任务的时候，根据yarn-session初始化的yarnDescription 把 flink 任务的jobGraph提交到yarn上去执行

60、同样的代码逻辑写在单独的main函数中就可以成功的消费kafka ，写在一个spring boot的程序中，接受外部请求，然后执行相同的逻辑就不能消费kafka。你遇到过吗？能给一些查问题的建议，或者在哪里打个断点，能看到为什么消费不到kafka的消息呢？

61、请问下flink可以实现一个流中同时存在订单表和订单商品表的数据 两者是一对多的关系 能实现得到 以订单表为主 一个订单多个商品 这种需求嘛

62、在用中间状态的时候，如果中间一些信息保存在state中，有没有必要在redis中再保存一份，来做第三方的存储。

63、能否出一期flink state的文章。什么场景下用什么样的state？如，最简单的，实时累加update到state。

64、flink的双流join博主有使用的经验吗？会有什么常见的问题吗

65、窗口触发的条件问题

66、flink 定时任务怎么做？有相关的demo么？

67、流式处理过程中数据的一致性如何保证或者如何检测

68、重启flink单机集群，还报job not found 异常。

69、kafka的数据是用 org.apache.kafka.common.serialization.ByteArraySerialize序列化的，flink这边消费的时候怎么通过FlinkKafkaConsumer创建DataStream？

70、现在公司有一个需求，一些用户的支付日志，通过sls收集，要把这些日志处理后，结果写入到MySQL，关键这些日志可能连着来好几条才是一个用户的，因为发起请求，响应等每个环节都有相应的日志，这几条日志综合处理才能得到最终的结果，请问博主有什么好的方法没有？

71、flink 支持hadoop 主备么？ hadoop主节点挂了 flink 会切换到hadoop 备用节点？

72、请教大家: 实际 flink 开发中用 scala 多还是 java多些？ 刚入手 flink 大数据 scala 需要深入学习么？

73、我使用的是flink是1.7.2最近用了split的方式分流，但是底层的SplitStream上却标注为Deprecated，请问是官方不推荐使用分流的方式吗？

74、KeyBy 的正确理解，和数据倾斜问题的解释

75、用flink时，遇到个问题 checkpoint大概有2G左右， 有背压时，flink会重启有遇到过这个问题吗

76、flink使用yarn-session方式部署，如何保证yarn-session的稳定性，如果yarn-session挂了，需要重新部署一个yarn-session，如何恢复之前yarn-session上的job呢，之前的checkpoint还能使用吗？

77、我想请教一下关于sink的问题。我现在的需求是从Kafka消费Json数据，这个Json数据字段可能会增加，然后将拿到的json数据以parquet的格式存入hdfs。现在我可以拿到json数据的schema，但是在保存parquet文件的时候不知道怎么处理。一是flink没有专门的format parquet，二是对于可变字段的Json怎么处理成parquet比较合适？

78、flink如何在较大的数据量中做去重计算。

79、flink能在没有数据的时候也定时执行算子吗？

80、使用rocksdb状态后端，自定义pojo怎么实现序列化和反序列化的，有相关demo么？

81、check point 老是失败，是不是自定义的pojo问题？到本地可以，到hdfs就不行，网上也有很多类似的问题 都没有一个很好的解释和解决方案

82、cep规则如图，当start事件进入时，时间00:00:15，而后进入end事件，时间00:00:40。我发现规则无法命中。请问within 是从start事件开始计时？还是跟window一样根据系统时间划分的？如果是后者，请问怎么配置才能从start开始计时？

83、Flink聚合结果直接写Mysql的幂等性设计问题

84、Flink job打开了checkpoint，用的rocksdb，通过观察hdfs上checkpoint目录，为啥算副本总量会暴增爆减

85、[Flink 提交任务的 jar包可以指定路径为 HDFS 上的吗]()

86、在flink web Ui上提交的任务，设置的并行度为2，flink是stand alone部署的。两个任务都正常的运行了几天了，今天有个地方逻辑需要修改，于是将任务cancel掉(在命令行cancel也试了)，结果taskmanger挂掉了一个节点。后来用其他任务试了，也同样会导致节点挂掉

87、一个配置动态更新的问题折腾好久（配置用个静态的map变量存着，有个线程定时去数据库捞数据然后存在这个map里面更新一把），本地 idea 调试没问题，集群部署就一直报 空指针异常。下游的算子使用这个静态变量map去get key在集群模式下会出现这个空指针异常，估计就是拿不到 map

88、批量写入MySQL，完成HBase批量写入

89、用flink清洗数据，其中要访问redis，根据redis的结果来决定是否把数据传递到下流，这有可能实现吗？

90、监控页面流处理的时候这个发送和接收字节为0。

91、sink到MySQL，如果直接用idea的话可以运行，并且成功，大大的代码上面用的FlinkKafkaConsumer010，而我的Flink版本为1.7，kafka版本为2.12，所以当我用FlinkKafkaConsumer010就有问题，于是改为 FlinkKafkaConsumer就可以直接在idea完成sink到MySQL，但是为何当我把该程序打成Jar包，去运行的时候，就是报FlinkKafkaConsumer找不到呢

92、SocketTextStreamWordCount中输入中文统计不出来，请问这个怎么解决，我猜测应该是需要修改一下代码，应该是这个例子默认统计英文

93、 Flink 应用程序本地 ide 里面运行的时候并行度是怎么算的？



