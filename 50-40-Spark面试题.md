[TOC]



# 1.知识篇



# 2.**开发调优**

## 2.1 对多次使用的RDD进行持久化

Spark中对于一个RDD的执行多次算子的默认原理是这样的：每次你对一个RDD执行一个算子的操作的时候，都会从源头计算一遍（因为RDD是根据finalStage递归往前找到第一个算子开始执行），计算出来那个RDD来，然后对这个RDD执行你的算子操作，这种方式的性能是很差的。

 cache机制是每计算出一个要cache的partition就直接将其cache到内存了，但是checkpoint没有直接使用这种第一次计算得到就存储的方法，而是等到job结束后另外启动专门的job去完成checkpoint，也就是说需要checkpoint的RDD会被计算两次，因此在使用rdd.checkpoint()的时候，建议加上rdd.cache(),这样第二次运行的job酒不用重新计算改rdd，直接读取cache写磁盘。

## 2.2 避免创建重复的RDD

通常来说，我们在开发一个Spark作业的时候，首先是基于某个数据源（比如Hive或者HDFS）创建一个初始的RDD,接着对这个RDD执行某个算子操作，然后得到下一个RDD，以此类推，循环往复，直到计算出最终我们需要的结果，在这个过程中，多个RDD会通过不同的算子操作，（比如map，reduce等）串联起来，这个RDD串就是，RDD linage，也就是RDD的血缘关系。

我们在开发过程中注意：对于同一份数据，只创建一个RDD,不能创建多个RDD来代表一份数据。

一些Spark初学者在刚刚开始开发Spark作业的时候，或者有经验的工程师在开发RDD linage及其冗长的Spark作业时候，可能忘记了自己之前对某一份数据已经创建过一个RDD了，从而导致对同一份数据，创建了多个RDD,这就一位置，我们的Spark作业会进行多次计算来创建代表相同数据的RDD，进而增加了作业的性能开销。

```java
// 错误的用户，针对同一份数据源创建多个RDD
val rdd1 = sc.textFile("hdfs://hadoop1:9000/hello.txt")
.....
rdd1.map()

val rdd2 = sc.textFile("hdfs://hadoop1:9000/hello.txt")
.....
rdd2.flatmap()
  
// 正确的用户，针对同一份数据源创建多个RDD
val rdd1 = sc.textFile("hdfs://hadoop1:9000/hello.txt")
.....
rdd1.map()
rdd1.flatmap()

// 但是注意，到这里我们的优化还没结束，为什么？因为rdd1被我们使用了2次，其实第二次使用的时候，
// 他会从源头重新开始计算，也就会带来性能的开销，这里就要用持久化操作了。
  
// 这里应该把这个RDD进行持久化，这样才能保证一个rdd呗多次使用才计算一次！！！

```

## 2.3 尽可能复用同一个RDD

除了要避免在开发过程中对一份完全相同的数据创建多个 RDD之外在对不同的数据执行算子操作时还要尽可能地复用一个RDD.比如说，有一个RDD的数据格式是key-value类型的。另-一个是单value类型的。这两个RDD的value数据是完全-样的。 那么此时我们可以只使用key-value类型的那个RDD ,因为其中已经包含了另-个的数据.对于类似这种多个RDD的数据有重叠或者包含的情况.我们应该尽量复用一个RDD。这样可以尽可能地减少RDD的数量。从而尽可能减少算子执行的次数。

```scala
错误的用法
// 我们有一份数据 . <Long,String>类型的数据rdd
JavaPairRDD<Long.String> rdd1 = ....
JavaRDD <String> rdd2=rdd1.map...
// 分别对rdd1和rdd2进行不同的算子的操作
rdd1.reduceBykl..
rdd2.map.....


//正确用法.其实我们这个例子里面. rdd1和rdd2只是数据格式不一 样而已. rdd2的数据是rdd1的一 个子集而已.
// 但是我们在开发的时候却创建了两个rdd .并对两个rdd进行了算子的操作。那么这个时候是增加了性能的开销。
//那么这种情况下我们完全可以复用同一 个rdd. .然后对这个rdd既做reduceBykey操作.又做我们需要的map操作。
JavaPairRDD<Long.String> rdd1 = ....
rddl.reduceBykey(.)
rdd1.map(tuple_2 ....)

//当然到这个我们确实是复用了同一个rdd了 .但是我们在执行第一个算子 操作的时候.是不是还是需要重新计算一次
// rdd1 的数据.那么这个时候是不是还是会需要到我们
// 开发调优的第一讲给大家讲的需要对rdd1进行持久化操作。 这个时候才能保证一个rdd被执行一 次。

```

## 2.4 尽量避免使用shuffle类算子

如果有可能的话要尽量避免使用shuffle类算子.因为Spark作业运行过程中最消耗性能的地方就是shuffle过程. shuffle过程，简单来说,就是将分布在集群中多个节点上的同-一个key ,拉取到同一个节点上,进行聚合或join等操作。比如reduceByKey. join 等算子。都会触发shuffle操作.

shuffle过程中,各个节点上的相同key都会先写入本地磁盘文件中,然后其他节点需要通过网络传输拉取各个节点上的磁盘文件中的相同key.而且相同key都拉取到同一个节点进行聚台操作时。还有可能会因为一个节点上处理的key过多,导致内存不够存放,进而溢写到磁盘文件中。因此在shuffle过程中。可能会发生大量的磁盘文件读写的IO操作,以及数据的网络传输操作。磁盘IO和网络数据传输也是shuffle性能较差的主要原因。

因此在我们的开发过程中,能避免则尽可能避免使用reduceByKey. join. distinct. repartition 等会进行shuffle的
算子,尽量使用map类的非shuffle尊子。这样的话。没有shufle操作或者仅有较少shuffle操作的Spark作业。可以大大减少性能开销。

spark中会导致shuffle操作的有以下几种算子
1. repartition 类的操作:比如repartition. repartitionAndSortWithinPartitions. coalesce 等

2. byKey 类的操作:比如reduceByKey. groupByKey. sortByKey 等

3. join类的操作:比如join. cogroup 等



重分区:一般会shuffle.因为需要在整个生群中,对之前所有的分区的数据进行随机。均匀的打乱.然后把数据放入下游新的指定数量的分区内

byKey类的操作:因为你要对一个key，进行聚合操作。那么肯定要保证集群中，所有节点上的。相同的key,一定是到同一个节点上进行处理

 join类的操作:两个rdd进行join ,就必须将相同join key的数据，shuffle到同一个节点上，然后进行相同key的两个rdd数据的笛卡尔乘积

```scala
//传统的join的操作
val rdd3=rdd1.join(rdd2) 
//这样的话肯定会发生shuffle的操作。因为要把相同的key拉取到一个节点上。然后进行join

// 我们用Broadcast+ map的操作代替join操作。不会发生shuffle的操作。
// 首先使用broadcast将一个 数据量较小的rdd作为广播变量(作为广播变量的rdd1数据量不宜超过1个G, 1G以内
// 我们认为没问题。)
val rdd2Data= rdd2.collect();
val rdd2DataBoradCast= sc.broadcat( rdd2Data)
//在rdd1.map算子中。可以从rdd2DataBoradCast中。获取到所有rdd2的数据。
// 然后进行遍历。如果发现rdd2中某条数据的key与rdd1的数据的当前key-样。那么我们就可以判断可以进行join操作。
// 然后自己根据自己的需求把结果给拼接起来( string。tuple )然后返回去
rdd1.map(rdd2DataBoradCast ...
         
```

## 2.4 **使用map-side预聚合的shuffle操作**

如果因为需要，一定要使用shuffle操作，无法用map类的算子来替代，那么尽量使用可以map-side预聚合的算子。

## 2.5 **使用高性能的算子**

1. 使用reduceByKey/aggregateByKey替代groupByKey
2. **使用mapPartitions替代普通map**
3. **使用foreachPartitions替代foreach**
4. **使用filter之后进行coalesce操作**
5. **使用repartitionAndSortWithinPartitions替代repartition与sort类操作**

## 2.6 **广播大变量**

有时在开发过程中，会遇到需要在算子函数中使用外部变量的场景（尤其是大变量，比如100M以上的大集合），那么此时就应该使用Spark的广播（Broadcast）功能来提升性能。

在算子函数中使用到外部变量时，默认情况下，Spark会将该变量复制多个副本，通过网络传输到task中，此时每个task都有一个变量副本。如果变量本身比较大的话（比如100M，甚至1G），那么大量的变量副本在网络中传输的性能开销，以及在各个节点的Executor中占用过多内存导致的频繁GC，都会极大地影响性能。

因此对于上述情况，如果使用的外部变量比较大，建议使用Spark的广播功能，对该变量进行广播。广播后的变量，会保证每个Executor的内存中，只驻留一份变量副本，而Executor中的task执行时共享该Executor中的那份变量副本。这样的话，可以大大减少变量副本的数量，从而减少网络传输的性能开销，并减少对Executor内存的占用开销，降低GC的频率。

## 2.7 **使用Kryo优化序列化性能**

在Spark中，主要有三个地方涉及到了序列化：

1. 在算子函数中使用到外部变量时，该变量会被序列化后进行网络传输（见“7.广播大变量”中的讲解）。
2. 将自定义的类型作为RDD的泛型类型时（比如JavaRDD，Student是自定义类型），所有自定义类型对象，都会进行序列化。因此这种情况下，也要求自定义的类必须实现Serializable接口。
3. 使用可序列化的持久化策略时（比如MEMORY_ONLY_SER），Spark会将RDD中的每个partition都序列化成一个大的字节数组。

对于这三种出现序列化的地方，我们都可以通过使用Kryo序列化类库，来优化序列化和反序列化的性能。Spark默认使用的是Java的序列化机制，也就是ObjectOutputStream/ObjectInputStream API来进行序列化和反序列化。但是Spark同时支持使用Kryo序列化库，Kryo序列化类库的性能比Java序列化类库的性能要高很多。

## 2.8 **优化数据结构**

Java中，有三种类型比较耗费内存：

* 对象，每个Java对象都有对象头、引用等额外的信息，因此比较占用内存空间。
* 字符串，每个字符串内部都有一个字符数组以及长度等额外信息。
* 集合类型，比如HashMap、LinkedList等，因为集合类型内部通常会使用一些内部类来封装集合元素，比如Map.Entry。

因此Spark官方建议，在Spark编码实现中，特别是对于算子函数中的代码，尽量不要使用上述三种数据结构，尽量使用字符串替代对象，使用原始类型（比如Int、Long）替代字符串，使用数组替代集合类型，这样尽可能地减少内存占用，从而降低GC频率，提升性能。

# 3.**资源调优**

Executor的内存主要分为三块：第一块是让task执行我们自己编写的代码时使用，默认是占Executor总内存的20%；第二块是让task通过shuffle过程拉取了上一个stage的task的输出后，进行聚合等操作时使用，默认也是占Executor总内存的20%；第三块是让RDD持久化时使用，默认占Executor总内存的60%。



# 4.**数据倾斜调优**

数据倾斜**只会发生在shuffle过程中**，可能会触发shuffle操作的算子：distinct、groupByKey、reduceByKey、aggregateByKey、join、cogroup、repartition等。出现数据倾斜时，可能就是你的代码中使用了这些算子中的某一个所导致的。

## 4.1 定位数据倾斜

1. 某个task执行特别慢的情况
2. 某个task莫名其妙内存溢出的情况
3. 查看导致数据倾斜的key的数据分布情况

## 4.2 **数据倾斜的解决方案**

1. 使用Hive ETL预处理数据
2. 过滤少数导致倾斜的key
3. 提高shuffle操作的并行度
4. 两阶段聚合（局部聚合+全局聚合）
5. 将reduce join转为map join
6. 采样倾斜key并分拆join操作
7. 使用随机前缀和扩容RDD进行join

# 5.**shuffle调优**

