# Spark RDD

## RDD基本

### Parallelized Collections

​	使用sparkcontext的parallelize方法可以从现有的集合创建RDD。这个方法可以指定分区数。

Spark会给节点的每个分区运行一个任务。

### 读取文件方法

##### textFile	

​	可以使用本地文件系统上的路径，该文件也必须可以在工作节点上的相同路径上，也可以把它复制到所有工作节点的相应路径，或使用网络共享文件系统

​	这个方法支持：一个目录、通配符、压缩文件。为文件中数据的分区的顺序，取决于对于文件从文件系统返回的顺序

​	这个方法有一个参数支持分区，Spark为文件的每个块创建一个分区（HDFS默认一个块128M），传递的分区参数数不能小于块数

##### wholeTextFiles

wholeTextFiles可以读取一个包含多个小文本文件的目录，并以（文件名，内容）的形式返回每个文件。这个方法也可以传递分区参数。

##### SequeneceFiles[K,V]

​	K和V是实现了Writable接口的子类。一些基本的类型，可以被直接读取为语言对应的基本类型，如IntWritable->Int

##### HadoopRDD



## RDD运算

首先注意以下的操作

```scala
var counter = 0
var rdd = sc.parallelize(data)

// Wrong: Don't do this!!
rdd.foreach(x => counter += x)

println("Counter value: " + counter)
```

​	在本地模式下，因为变量在一个JVM中，counter确实是可以被更新的，但是在集群模式下，foreach这个转换行为会被发送个每个节点，每个节点上的JVM都有一个counter变量的副本，他们都在自己的JVM内存上进行累加操作，最终的结果不会是正确的。

​	对于这种操作，用到Spark提供的累加器，Accumulators

##### 转换操作

| **map**(*func*)                                              | 对每个元素执行func操作返回新的RDD                            |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| **filter**(*func*)                                           | 过滤器（要true的）                                           |
| **flatMap**(*func*)                                          | 每个输入项进行func操作将返回0、多个输出，所以func应返回一个Seq。flatMap的最终返回值为RDD[T]，Map函数可能就是RDD[Seq[T]] |
| **mapPartitions**(*func*)                                    | map在每分区上单独执行，func返回应该是可迭代的集合            |
| **mapPartitionsWithIndex**(*func*)                           | 同mapPartitions，但是func参数多一个，为分区索引              |
| **sample**(*withReplacement*, *fraction*, *seed*)            | 随机取样                                                     |
| **union**(*otherDataset*)                                    | 将两个相同类型的RDD连接起来                                  |
| **intersection**(*otherDataset*)                             | 取两个RDD的交集                                              |
| **distinct**([*numPartitions*]))                             | 去重                                                         |
| **groupByKey**([*numPartitions*])                            | shuffle然后用key分组聚合（如果要对分组后的数据进行其他操作，使用reduceByKey或aggregateByKey性能会更好；输出的并行度取决于分区的数量，你也可以指定分区数量） |
| **reduceByKey**(*func*, [*numPartitions*])                   | 不一定shuffle，对同一key的元素执行操作                       |
| **aggregateByKey**(*zeroValue*)(*seqOp*, *combOp*, [*numPartitions*]) | seqOp对一个分区内的累积操作、combOp对两个分区进行的合并操作，zeroValue是合并操作时使用到的中间值 |
| **sortByKey**([*ascending*], [*numPartitions*])              | 排序                                                         |
| **join**(*otherDataset*, [*numPartitions*])                  | `leftOuterJoin`, `rightOuterJoin`, and `fullOuterJoin`.      |
| **cogroup**(*otherDataset*, [*numPartitions*])               | 对K,V和K,W的两个RDD进行聚合(K,(Iterable<V>,Iterable<W>))     |
| **cartesian**(*otherDataset*)                                | 相同长度的RDD的拉链操作                                      |
| **pipe**(*command*, *[envVars]*)                             | 通过Bash命令管理RDD分区。RDD的元素被写入程序的标准输入，命令得到的标准输出将被包装成RDD返回 |
| **coalesce**(*numPartitions*)                                | 重分区操作（分区不平衡）                                     |
| **repartition**(*numPartitions*)                             | 分区平衡的重分区操作（Shuffle）                              |
| **repartitionAndSortWithinPartitions**(*partitioner*)        | 重分区并在分区内排序                                         |

##### 计算操作

| **reduce**(*func*)                                 | 对RDD的相邻元素进行func操作                                  |
| -------------------------------------------------- | ------------------------------------------------------------ |
| **collect**()                                      | 收集为集合                                                   |
| **count**()                                        | size                                                         |
| **first**()                                        | take(1)                                                      |
| **take**(*n*)                                      | 获取RDD中的第N个元素                                         |
| **takeSample**(*withReplacement*, *num*, [*seed*]) | Return an array with a random sample of *num* elements of the dataset, with or without replacement, optionally pre-specifying a random number generator seed. |
| **takeOrdered**(*n*, *[ordering]*)                 | 返回自然序列或自定义比较器的RDD中的第一组n个元素             |
| **saveAsTextFile**(*path*)                         | 保存为textfile                                               |
| **saveAsSequenceFile**(*path*) (Java and Scala)    | 保存为hadoop sequence file                                   |
| **saveAsObjectFile**(*path*) (Java and Scala)      | Write the elements of the dataset in a simple format using Java serialization, which can then be loaded using `SparkContext.objectFile()`. |
| **countByKey**()                                   | 计算相同K的值个数=>(k,Int)                                   |
| **foreach**(*func*)                                | 遍历                                                         |

### Shuffle操作

​	某些运算会涉及Shuffle操作，Shuffle操作涉及跨节点的数据传输、磁盘IO，因此Shuffle操作非常昂贵。

​	可能引起Shuffle的操作有：reparations，类似caolesce，ByKey操作Join操作。

​	Shuffle操作会在磁盘上产生大量的临时文件。在Spark1.3开始，这些文件被保存下来，直到RDD不再被使用并被垃圾回收之后，临时文件才会被清理。这样做是为了重新计算时不在需要重新生成中间临时文件。如果RDD一直被保存，GC就会很长时间不被启动，那么对内存的开销将非常大

### 持久化

​	cache或persist操作会将中间RDD结果保存在内存中，这对迭代算法和快速启动非常有用，性能往往可以提升很多。

##### 	persist级别

| Storage Level                          | Meaning                                                      |
| :------------------------------------- | :----------------------------------------------------------- |
| MEMORY_ONLY                            | 这是默认级别。会将Java Object存储在JVM中，若内存放不下的，则不会被缓存，需要时再重新计算 |
| MEMORY_AND_DISK                        | 将Java Object存储在JVM中，若内存放不下，则存储在磁盘上。     |
| MEMORY_ONLY_SER (Java and Scala)       | 将RDD序列化存储在内存，这要比不序列化的空间效率更高，只是在反序列化时耗费CPU（内存存不下的，需要时再执行重计算） |
| MEMORY_AND_DISK_SER (Java and Scala)   | 与上一个类似，只是将存不下的放在磁盘上                       |
| DISK_ONLY                              | 只将RDD存在磁盘上                                            |
| MEMORY_ONLY_2, MEMORY_AND_DISK_2, etc. | 与上面的级别一样，只是会在两个节点上复制每个分区             |
| OFF_HEAP (experimental)                | 与MEMORY_ONLY_SER 级别相同，只是将数据存储在堆外内存（需要开启堆外内存） |

​	再Shuffle过程中，即使没有调用persist，可能也会对数据自动进行持久化，这是为了在避免shuffle过程中某些节点出现故障，还需要重新计算

##### 应该怎样选择持久化级别

- 对于Memory Only，性能最高，但是需要更多的内存
- 如果内存稍微小一点，可以使用效率高的序列化库，这样会增加一些CPU开销
- 除非计算函数代价非常高，否则尽量不要把数据溢出到磁盘。因为重新计算要比从磁盘读取更快
- 如果你想要快速的故障恢复，使用复制级别持久化，这样就不用等待重新计算，但是会需要更多的内存空间

##### 持久化数据的释放

​	Spark自动监控每个节点上的持久化数据，用LRU算法释放持久化的数据。如果想手动释放，可以调用RDD.unpersist方法，这个方法默认是异步的，可以将blocking参数设为true

### 共享变量

##### 广播变量

```scala
scala> val broadcastVar = sc.broadcast(Array(1, 2, 3))
broadcastVar: org.apache.spark.broadcast.Broadcast[Array[Int]] = Broadcast(0)

scala> broadcastVar.value
res0: Array[Int] = Array(1, 2, 3)
```

​	广播变量允许程序员在每个节点上保存一个只读变量的缓存。例如它可以被用来给每个节点一个大型输入数据集副本。Spark使用高效的广播算法来分配广播变量，以减少通信成本。

​	Spark的action是通过一组阶段执行的，由分布式的shuffle操作分割。Spark会自动广播每个阶段内的任务所需的共同数据。以这种方式广播的数据以序列化的形式被缓存，并在运行每个任务之前被反序列化。**<u>也就是说，只有当多个阶段（transformation、action）操作需要用到共同的数据或以反序列化的形式缓存数据存储数据很重要时，明确的创建broadcast才最有用</u>**

- 在广播变量被创建后，集群运行的任何函数都应该使用它而不是在函数中使用一个val值，这样v就不会被多次运到节点上
- 广播变量不可变
- 要释放广播变量复制到执行程序的资源，调用unpersist。如果之后再次使用广播，则会重新广播。
- 要永久释放调用destory（方法默认非阻塞，可以设置blocking=true为阻塞方式），则广播变量会被释放

##### 累加器

​	累加器常被用于计数和求和操作，它提供了add操作

```scala
scala> val accum = sc.longAccumulator("My Accumulator")
accum: org.apache.spark.util.LongAccumulator = LongAccumulator(id: 0, name: Some(My Accumulator), value: 0)

scala> sc.parallelize(Array(1, 2, 3, 4)).foreach(x => accum.add(x))
...
10/09/29 18:41:08 INFO SparkContext: Tasks finished in 0.317106 s

scala> accum.value
res2: Long = 10
```

​	除此之外的基本类型，还可以自定义累加器，这需要实现AccumulatorV2，重写如reset、add方法。而且自定义累加器的执行add操作之后所得结果可以与原类型不同

​	注意：

- 累加器的操作只能在action方法中更新
- Spark会保证每个任务只更新对累加器的更新操作被执行一次，重启任务将不会被再次更新。
- 在transformation操作中，累加器的更新操作是不止一次的，如果任务重新执行，他也会被重新更新

## 部署测试



### 部署到集群（application submission）

​	打包应用程序，将hadoop、spark的依赖项作为provide，然后组装好jar包将其交给集群spark-submit运行任务

##### 基本部署 spark-submit

​	

```bash
./bin/spark-submit \
  --class <main-class> \
  --master <master-url> \
  --deploy-mode <deploy-mode> \
  --conf <key>=<value> \
  ... # other options
  <application-jar> \
  [application-arguments]
```

​	注意conf里面的键值对写成 "key=vaule"，以下列出常用参数

-  --master：
  - 常用yarn
  - spark://host:port
  - local一个工作线程、local[K]k个工作线程、local[K,F]F是指定任务连续错误时达到F次退出、local[*]客户机逻辑线程数、local-cluster[N,C,M]本地集群测试模式N个工作程序、每个工作程序有C个core，每个工作程序有M MB内存
- --deploy-mode
  - 本地 local
  - 集群 cluster
- --class main class
- --name spark app名称
- --jars 逗号分隔提供的运行时jar包
- --executor-memory 执行器内存
- --num-executors ：执行器启动的核心数
- --total-executor-cores 所有执行器的总核心数
- --supervise 错误非0退出 重新启动
- file

### 启动Spark Job

​	spark.launcher包提供了启动Spark Job作为子进程的工具

### 单元测试

​	只需在你的测试中创建一个SparkContext，将url设置为本地，然后运行操作，最后调用stop方法关闭context。





​	