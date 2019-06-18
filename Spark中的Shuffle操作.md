
### 什么是Shuffle
shuffle是所有MapReduce计算框架必须经过的阶段，用于打通Map阶段的输出和Reduce阶段的输入：Map阶段的中间输出结果按照key的哈希值分发到不同的Reduce任务中。

Spark中，Shuffle可以理解为repartition，即数据的重新分区。是否发生Shuffle也是区分RDD之间的依赖关系是宽依赖还是窄依赖的标准。

### Shuffle的基本过程
![Shuffle: write phase & read phase](../images/2019/05/shuffle-write-phase-read-phase.png)

Shuffle基本过程如上图，Shuffle涉及到两部分操作：Map端将临时输出结果根据key的哈希值分到不同的bucket中，Reduce端将负责的bucket数据取回本地继续执行任务。

Shuffle涉及到数据的排序、序列化、磁盘IO、网络传输等一系列工作，因此Shuffle是一项开销很大的工作。Spark对Shuffle的过程做了一系列的优化，下面看一下Spark中的Shuffle演进过程。


### 几种shuffle类型
#### 1. 早期版本的Hash Based Shuffle
![basic hash shuffle](../images/2019/05/basic-hash-shuffle.png)

1. map任务会为每个Reduce任务创建一个bucket，即分区数为ReduceTask的数目
2. Map任务将中间结果根据分区函数写入到不同的bucket中
3. Reduce任务获取负责的bucket中的数据，作为数据的输入

为减少随机IO的次数，Spark会首先将数据写入到内存的一个buffer中，达到足够大小后再一次性的写入到文件系统中。buffer的大小由配置参数'spark.shuffle.file.buffer.kb'控制。

在这个版本的Shuffle中有一个问题：假设有M个Map任务，R个Reduce任务，那么一共会产生M*R个buffer。当M和N很大时，会产生大量的内存占用。因此在提交[SPARK-2503](https://issues.apache.org/jira/browse/SPARK-2503)中，将buffer的默认大小从100K减少到了32K。

#### 2. 引入Consolidate shuffle files机制
上面的解决内存占用过多的问题的方法是降低buffer的默认大小，但根本问题还是没有解决，内存占用的大小还是`M*R*buffer_size`。因此在提交[SPARK-751](https://issues.apache.org/jira/browse/SPARK-751)又做了一次改进：在Executor级别，而不是MapTask级别执行MapShuffle。如下图：
![consolidate shuffle](../images/2019/05/consolidate-shuffle.png)

在一个Executor中，每个MapTask的输出都对应到一个File的一个Segment。此时，一个文件内Segment的数量对应MapTask的数量，文件数量对应ReduceTask的数量R。文件数量等于R，意味着buffer的数量由M*R减少到了R。

要开启Consolidate shuffle file需要开启配置`spark.shuffle.consolidateFiles`，但是目前这种shuffle方式已经不再使用了，因此这个配置项也就没意义了。

#### 3. The sort based shuffle
在hash shuffle中引入了consolidate机制后，Spark将每个Executor中文件的个数从M*R减少到了R个。但是在数据量很大的情况下，意味着Reduce任务的数量也很大，分区的数量很多，造成每个Executor中的文件数量还是很多。在这个时候，Spark对于很大规模数据的处理的Shuffle过程就会很吃力。

这时，Spark参考Hadoop的实现：一个MapTask对应一个文件，不同的分区对应文件的不同Segment，如下图。

![sort based shuffle](../images/2019/05/sort-based-shuffle.png)

在单个Executor中，每个MapTask产生一个数据文件和一个索引文件。数据文件包括所有的partition的数据，一个partition对应文件中的一个Segment；索引文件记录Segment的位置以及对应关系。图中是一个Executor中指定并行度为3，因此一次性会有3个同时运行的MapTask，此时只会产生3个数据文件，即使是在整体的数据量很大的情况下。

### Spark中Shuffle的演进
- Spark0.8之前
  默认使用Hash Based Shuffle。通常会产生大量的文件，伴随着大量的随机磁盘IO以及大量的内存开销
- Spark0.8.1
  为Hash Based Shuffle引入File Consolidate机制。通过文件合并，为每个ReduceTask产生一个文件，将文件数目从M*R减少到了R
  [SPARK-751](https://issues.apache.org/jira/browse/SPARK-751)
- Spark1.1.0
  引入sort Based shuffle。参考Hadoop的实现，Executor中的每个MapTask产生一个文件(以及一个索引文件)，大大减少了单个Executor中的临时文件的数量。但此时默认的仍然为Hash Based Shuffle。
  [SPARK-2045](https://issues.apache.org/jira/browse/SPARK-2045)
- Spark1.2
  默认的shuffle方式改为Sort Based Shuffle
- Spark1.4
  引入Tungsten-Sort Based Shuffle
- Spark1.6
  Tungsten-Sort并入Sort Based Shuffle
- Spark2.0
  弃用Hash Based Shuffle，从此Spark的shuffle只剩下Sort Based Shuffle

### Spark Shuffle源码结构
最上层是`org.apache.spark.shuffle.ShuffleManager`，目前唯一的实现只剩下了`org.apache.spark.shuffle.sort.SortShuffleManager`。`SortShuffleManager`有两种类型：sort和Tungsten-Sort。

调用流程：在`org.apache.spark.scheduler.ShuffleMapTask#runTask`中根据用户配置获取`ShuffleManager`。在该类型的`ShuffleManager`上调用`getWriter`以及`write`方法。


### Spark Shuffle相关配置

- `spark.shuffle.compress`
  是否压缩map的输出文件，默认为true
- `spark.shuffle.file.buffer`
  为每个输出文件流创建的缓冲区的大小，用于减少IO的次数。默认为32K。
- `spark.shuffle.manager`
  指定哪种sort manager，这个配置似乎没用，在代码中，无论指定sort还是tungsten-sort，最终都是产生一个SortShuffleManager类。
- `spark.shuffle.io.maxRetries`
  - 默认值 3
  - 说明：Shuffle read时由于IO异常导致的读取失败，会自动进行retry的次数
- `spark.shuffle.io.retryWait`
  - 默认值：5
  - 说明：Shuffle read时由于IO异常导致读取失败时，自动进行retry的时间间隔
-

### 参考资料
[Understand the Shuffle Component in Spark-core](https://trongkhoanguyen.com/spark/understand-the-shuffle-component-in-spark-core/)
[Spark Shuffle技术演进](https://www.jianshu.com/p/4c5c2e535da5)

### 待看
[Shuffle内存分析](https://www.jianshu.com/p/c83bb237caa8)
