# MapReduce 核心知识

## 输入输出

​	(input) `<k1, v1> ->` **map** `-> <k2, v2> ->` **combine** `-> <k2, v2> ->` **reduce** `-> <k3, v3>` (output)

​	key所使用的类必须实现WriteableComparable接口，以此来支持排序。

## MapReduce 用户接口

### Mapper

- MapReduce会为每个InputSplit生成一个Map任务
- 你可以覆写cleanup方法来执行任何清理工作
- 用户可以指定Comparator来控制分组（控制key）
- Mapper输出经过排序，然后按每个Reduce进行分区。用户可以实现Partitioner接口来指定哪些键去哪个Reducer
- 用户可以指定一个combiner控制中间值的组合，这样可以减少中间值向Reducer任务过度时产生的数据传输的数据量（控制Value）。
- 中间的（排序后的输出）总是以（k-l,k,v-l,v)格式化存储，应用程序可以通过CompressionCodec配置怎样压缩。

##### Map任务有多少个？

Map任务的数量默认是取决于输出文件的规模，每个块默认是128MB，如果一个10TB的数据，会被分为81920块，因此也就会有81920个Map任务

​	每个节点大约有10-100个Map任务，即使是轻量级的任务也有可能设置很多Map任务，当然你也可以自己指定Map任务数量。



### Reducer

- Reducer将Map产生的中间值进行排序、聚合，最终转换为另一键值对的输出。

- Reduce有三个阶段：Shuffle Sort reduce

  - Shuffle：这个阶段，框架通过HTTP来获取所有Map任务输出的相关分区。也就是这个阶段的输入是Map任务的排序输出

  - Sort：此阶段，框架对（Map之后的）输入进行分组（因为不同的Map可能输出相同的Key）

    注意：Sort和Shuffle同时发生，在获取Map输出时，他们会被合并

    - 二次Sort：如果想让分完组后的Key的输入和进行reduce时要求的Key输入不一样，可以使用Job.setSortComparatorClass指定二次分组。Job.setGroupingComparatorClass也可以控制中间key的分组方式，因此可以结合这个来模拟二次Sort

  - Reduce：注意Reduce的输出没有排序，默认只有一个Reduce任务，可以自己指定。Reduce任务应该指定为多少？ 0.95或1.75乘以节点数

    - 0.95时所有的Reduce都可以立即启动，并在Map完成时，传输Map的输出
    - 1.75时更快的节点可以先于其他的节点进行第一轮任务，然后与其他节点一起进行第二轮任务，可以实现更好的负载均衡。

    增加Reduce任务数量会增加节点的负载，但是有利于负载均衡和更快的故障恢复

    小的缩放因子可以为框架中为推测任务和失败任务保留一些可执行reduce的节点

    Reduce任务数量甚至可以指定为0，若这样做，将跳过Reduce的三个步骤，直接将Map任务的结果写入磁盘

### Partitioner 分区器

​	控制中间输出的Key的分区（分组），分区数与Reduce任务数一致

### Counter

​	counter是报告MapReduce运行统计信息的工具，Mapper和Reducer实现可以使用Counter来报告统计信息。Hadoop MapReduce默认绑定了一个Mapper、Reducer、Partitioner库



### Job Configuration

- Job用来配置MapReduce job的配置信息，有一些不可变的配置信息已被管理员设置好。
- Job常用来指定Mapper、combiner（如果有）、Partitioner、Reducer、InputFormat、OutputFormat
- 可选的：
  - 使用Comparator、要放入的分布式缓存的文件
  - 指定是否要压缩中间/作业的最终输出
  - 是否可以压缩作业以推测的方式执行
  - 每个任务的最大重试次数
- 用户可以通过Configuration类来获取配置信息

### 任务执行和环境

​	每个Map或Reduce任务都是一个JVM的子进程，可以通过`mapreduce.{map|reduce}.java.opts`指定运行时参数，它仅仅指定了任务的参数，而非其他守护进程的参数

##### 内存管理

​	`mapreduce.{map|reduce}.memory.mb` 可以指定任务的（递归创建的）最大虚拟内存值，单位是MB。该值必须大于等于JVM指定的Xms

##### Map 参数

​	Map产生的记录会首先被序列化存入缓冲区，然后再将元数据存储在记账缓冲区（指定大小的），如果这两个缓冲区哪个超过了阈值，Map任务将被阻塞，缓冲区内容将会再后台被排序存入磁盘，Map继续输出。Map完成后，剩余的记录写入磁盘，。缓冲区可以调整大小，更大的缓冲区意味速度更快，但是任务可用的内存将会变少。

| Name                             | Type  | Description                                                  |
| :------------------------------- | :---- | :----------------------------------------------------------- |
| mapreduce.task.io.sort.mb        | int   | 序列化和记账缓冲区的总大小                                   |
| mapreduce.map.sort.spill.percent | float | 序列化缓冲区的软限制，一旦超出这个值，线程将后台的将数据写入磁盘 |

​	阈值只是限制了超出某个值时候，应该将数据写入磁盘，不是到了阈值就会被阻塞，（类似一条指令只能经过完整的执行周期之后才能执行中断）如果此时向缓冲区写数据的操作还没有完成，则先继续写入，写完本条数据，再执行达到阈值的操作。

​	如果一条记录直接就大于缓冲区，则直接触发溢出操作，这个数据会被单独写入一个文件。

##### Shuffle/Reduce 参数

​	Reduce会通过Http将Partitioner分配给他的分区拉取到内存，并定期将这些输出写入磁盘。如果开启Map输出压缩选项，则每个输出都将压缩至内存中。以下选项会影响Reduce之前这些合并到磁盘的频率以及Reduce期间分配给Map输出的内存。

| Name                                          | Type  | Description                |
| :-------------------------------------------- | :---- | :------------------------- |
| mapreduce.task.io.soft.factor                 | int   | 指定磁盘上要同时合并的段数 |
| mapreduce.reduce.merge.inmem.thresholds       | int   |                            |
| mapreduce.reduce.shuffle.merge.percent        | float |                            |
| mapreduce.reduce.shuffle.input.buffer.percent | float |                            |
| mapreduce.reduce.input.buffer.percent         | float |                            |

##### 配置参数

| Name                       | Type    | Description                                    |
| :------------------------- | :------ | :--------------------------------------------- |
| mapreduce.job.id           | String  | job id                                         |
| mapreduce.job.jar          | String  | job.jar location in job directory              |
| mapreduce.job.local.dir    | String  | The job specific shared scratch space          |
| mapreduce.task.id          | String  | The task id                                    |
| mapreduce.task.attempt.id  | String  | The task attempt id                            |
| mapreduce.task.is.map      | boolean | Is this a map task                             |
| mapreduce.task.partition   | int     | The id of the task within the job              |
| mapreduce.map.input.file   | String  | The filename that the map is reading from      |
| mapreduce.map.input.start  | long    | The offset of the start of the map input split |
| mapreduce.map.input.length | long    | The number of bytes in the map input split     |
| mapreduce.task.output.dir  | String  | The task’s temporary output directory          |

##### 任务日志

​	任务日志存储在`${HADOOP_LOG_DIR}/userlogs`

##### 分布式库

### 任务提交和监控

​	Job提供了提交作业、获取任务组件的报告和日志、获取MapReduce节点的状态信息，Job 提交进程涉及：

1. 检查作业指定的输入输出
2. 计算作业的InputSplit值
3. 如果有必要，为作业的DistributionCache设置必要的记账信息
4. 分发作业，将jar复制到HDFS的MR系统目录

​	可以使用`mapred job -history all output.jhist`查看作业日志信息

##### 作业控制

​	用户可能需要链接MR来完成复杂的单个MR完成不了的任务，这样做任务的成功与否由用户自己负责，这种情况下，作业的控制选项是：

- job.submit()将作业提交到集群并立即返回
- job.waitForCompletion(boolean)将作业提交到集群并等待他完成

### 任务输入

​	`InputFormat`（子类FileInputFormat）描述了作业输入的信息，MR依赖`InputFormat`来完成：

- 验证输入的规范
- 将文件拆分为逻辑InputSplit实例，然后将每个实例分配给单独的Mapper
- 提供RecordReader实现，用于从逻辑InputSplit收集输入记录以供Mapper处理

​	默认的InputSplit拆分的文件块大小是FS指定的默认块大小，但是可以用`mapreduce.input.fileinputformat.split.minsize`指定拆分大小的下限。

​	如果使用的是TextInputFormat而不是FileInputFormat，则MR会检测带有.gz扩展名的文件，并使用适当的CompressCodec解压他们，这样的文件不可拆分，并且一个文件只能对应一个Mapper

##### InputSplit

​	InputSplit表示要被一个Mapper处理的数据，通常InputSplit是一个面向字节的输入视图，RecordReader则负责处理和呈现。

​	FileSplit是默认的InputSplit，他将`mapreduce.map.input.file`设置为逻辑拆分的输入文件的路径

##### RecordReader

​	RecordReader从InputSplit读取<K,V>，他为Mapper提供数据，也承担了处理边界的责任

### 任务输出

​	MR 依赖作业的OutputFormat来：

- 验证作业输出规范
- 提供用于写入作业输出文件的RecordWriter实现，输出文件存储在HDFS上

​	TextOutputFormata是OutputFormat的默认实现

##### OutputCommitter

​	OutputCommitter描述了MR作业的任务输出的提交，OutputCommitter作用：

1. 初始化期间设置作业。（例如创建临时输出目录，当作也处于PREP状态和初始化任务之后，作业设置由单独的任务完成，设置完任务后，作业将移至RUNNING状态）
2. 作业完成后清理作业（如清理临时输出目录，清理完成后，作业被声明为SUCCEDED/FAILED/KILLED）
3. 设置任务临时输出
4. 检查任务是否需要提交（避免提交过程）
5. 任务完成后提交到输出
6. 放弃任务提交（如果任务执行失败/终止，输出被清理。如果无法清理（在异常块中，将使用相同的重试ID启动单独的任务执行清理））

​	FileOutputCommitter是默认的OutputFormat，作业setup/clean任务会占用Map或reduce容器。JobCleanUp、TaskCleanup、JobSetup任务具有最高优先级，并且按顺序排列



##### Task Side-Effect Files 任务副作用文件 ？？？

​	如果想要创建或写入的文件与作业的输出文件不是一个（单独使用一个文件），如果再Mapper和Reducer同时运行时，对这同一个文件的访问会出现问题。这种情况应用程序编写者必须为每个任务尝试选择唯一的名称（例如使用attempid）

​	为了避免此类问题，当OutputCommitter时FileOutputCommitter时，MR维护了一个特殊的`${mapreduce.output.fileoutputformat.outputdir}/_temporary/_${taskid} `子目录，可以通过`${mapreduce.task.output.dir} `访问对于存储attempt_task输出的FS上的每个attempt_task，成功完成attempt_task后，`${mapreduce.output.fileoutputformat.outputdir}/_temporary/_${taskid}`将被提升为`${mapreduce.output.fileoutputformat.outputdir}`，框架会丢弃失败的attempt_task的子目录。这个过程对程序是完全透明的。





##### RecordWriter

​	RecordWriter将最终的<K,V>输出到最终文件

### 其他有用的功能

##### 提交作业给队列

##### Counters

##### 分布式缓存

##### Profiling

##### 调试

##### 数据压缩

##### 跳过坏的记录



# MapReduce应用案例

### 序列化：自定义Bean

​	Hadoop传输Bean时只可以传输实现了Writable接口的类型，因此想要传输其他类型的Bean需要实现Writable接口，实现时需要注意

- 如果将bean作为Key值，还需要实现Comparable接口
- 必须有一个空参构造方法，供给反序列化时使用
- 重写write和readFields方法，**<u>先write（序列化）哪个字段就先read（反序列化）哪个字段</u>**
- 如果向把bean作为结果输出到最终结果，需要为bean重写toString方法

### 按需将Key值放入不同的分区：自定义Partitioner

​	如果想要把key值再次划分，按需求写入不同的分区，那么可以自定义分区器，继承Partitioner重写getPartition方法

- getPartition方法提供三个参数，分别是Map输出的key和value，还有一个是总的分区数
- 最终要在Job中指定自定义的分区器，指定分区个数（有几个分区意味有几个Reduce task）

### Combiner 提前处理Map输出：微型Reducer

​	在Map得到输出后，在Map任务后先对值提前聚拢，这样需要传输的数据量就会变小，可以节省Reduce时进行远程数据拉取时产生的网络流量，也就降低了网络负载。不使用Combiner可能会发生：

- Reducer任务比较少如果需要处理大量的数据，时间可能比较少。
- Reducer需要远程拉取更多的数据，造成网络负载加重，阻塞时间更长

​	使用了Combiner：

- 由于Map任务较多，所以在Map之后对输出进行一次加工，可以使数据量变小



### 对最终的输出进行调整（例如：按Key分文件输出）：重写RecordWriter

### ReducerJoin：多表Join参照操作



### MapperJoin：多表Join参照操作

### ETL：数据转换任务，可以不设置Reducer

### 压缩文档进行运算

​	压缩可以节省带宽，节省磁盘，但是增加CPU开销。因此：

- 运算密集型，少用压缩
- IO密集型，可以用压缩

​	压缩类型，Bzip、Gzip、Snapy、LOZ，MR有三个地方可以设置压缩：输入端、MR中间、输出端

- Mapper输入端：考虑是否需要切片
- Mapper输出到Reducer输入中间：压缩和解压速度
- Reducer输出输出端：压缩率

# Mapred CLI



# 加密 Shuffle



# Shuffle/Sort 自定义

​	

# 分布式缓存部署新版本的MapReduce



# 使用Yarn支持的共享缓存
