# SparkStreaming编程指南之性能优化

## 性能优化
运行在集群上的SparkStreaming程序要想达到最佳性能需要一定的优化。本节介绍一些能够提升应用程序性能的参数和配置。总体来说，需要考虑两点：
1. 通过有效利用集群资源，来减少单批次数据的处理时间
2. 设置合理的批次大小，使得收到的数据能够尽快的被处理掉(数据处理速度与数据生成速度保持一致)

### 1. 减少批次处理时间
Spark中有许多优化手段能用来降低批次处理时间。这方面在[优化指南](https://spark.apache.org/docs/latest/tuning.html)中有详细的介绍。这一节列举其中最重要的几个。

#### 1.1 数据接收的并行度
从网络(如Kafka、Flume、socket等)接收数据，需要对数据进行反序列化，然后存放到Spark中。如果数据接收称为系统的瓶颈，可以考虑将数据接收并行化。注意，每个DStream都会创建一个单独的接收器(工作在worker机器节点上)，这个接收器负责接收一个单独的数据流。因此，可以创建多个输入DStream，每个DStream负责接收数据源中一个分区(shard或partition)的数据，从而达到进群接收多个数据流的数据的目的。例如，一个接收Kafka数据源两个topic的DStream，可以被拆分为两个，每个DStream消费Kafka的一个topic。这样就会运行两个receiver，并行的接收数据，从而提高整体的吞吐量。同时，我们还可以把多个DStream联合起来，创建一个联合Stream。所有能在单个DStream上进行的转换操作都可以被用在这个联合DStream上。联合DStream通过这样创建：

```Scala
val numStreams = 5
val kafkaStreams = (1 to numStreams).map { i => KafkaUtils.createStream(...) }
val unifiedStream = streamingContext.union(kafkaStreams)
unifiedStream.print()
```
另一个需要考虑的参数是接收器(receiver)的阻塞间隔(block interval)，block interval使用配置参数`spark.streaming.blockInterval`决定。对大部分receiver来说，接收到的数据都会在存放进Spark内存之前合并成数据块(data block)。每个批次中数据块的数量决定了对批次RDD执行如map操作类似的转换操作时会执行的task的数量(一个批次即一个RDD，数据块的多少即RDD的分区数目，每个分区上都会执行一个task)。例如，200ms的block interval会在一个2s的batch上创建10个task(2*1000/200，即RDD中有10个分区)。如果task的数量太少(少于所有Executor的core的总数)，那么会由于core的利用不充分造成效率的不足。在一个给定的batch interval下，想要增大task的数目，可以降低block interval。不过建议block interval最低不能低于50ms，不然任务启动的开销会成为一个问题。

另一个对数据进行分区的方法就是显式的调用`inputStream.repartition()`方法。这个方法会将输入数据分区成指定的数目。

#### 1.2 数据处理并行度
在计算过程中的任一个stage中，如果并发的task的数量不够高，也会导致集群资源利用率的低下。例如，分布式的聚合操作，如`reduceByKey`和`reduceByKeyAndWindow`，默认的并行task数量由`spark.default.parallelism`参数配置。可以通过传参来修改这个参数，也可以在修改配置文件中修改默认值。

#### 1.3 序列化
通过调整序列化的格式能大幅度降低序列化的开销。在Streaming程序中，有两种被序列化的数据。
- **输入数据**，默认情况下，receiver收到的数据以存储级别`StorageLevel.MEMORY_AND_DISK_SER_2`保存在Executor的内存中。为减少GC，收到的数据会被预先序列化为字节，并且数据会被备份以防Executor出错。这种存储级别决定了，数据会优先存放到内存中，只有在内存不足时，才会溢写到磁盘上。序列化的过程明显会有开销：接收器首先需要把收到的数据进行反序列化，然后使用Spark的序列化格式重新对数据进行序列化。
- **(Streaming算子产生的)持久化RDD**。Streaming计算过程中产生的RDD可能会被持久的存放在内存中。例如，由于窗口数据可能会被多次处理，因此在window操作中，窗口数据会被持久化在内存中。不同的是，SparkCore默认使用存储级别`StorageLevel.MEMORY_ONLY`，而Streaming算子中产生的持久化RDD使用存储级别`StorageLevel.MEMORY_ONLY_SER`(即序列化过后的)，目的是用来减少GC的消耗。

无论哪种情形，都可以使用Kryo序列化来减少CPU和内存消耗。查看[Spark优化](https://spark.apache.org/docs/latest/tuning.html#data-serialization)来了解更多细节。对于Kryo，可以考虑注册自定义类型，以及禁用对象引用跟踪(可以在配置指南中查看与[Kryo相关的配置](https://spark.apache.org/docs/latest/configuration.html#compression-and-serialization))

在一些特定情况下，如果数据量不是很大，可以考虑不使用序列化格式，但要注意取消序列化是否会引起额外的GC开销。例如，如果批次的间隔较短(几秒钟)，并且没有使用窗口操作，那么可以考虑通过显式的设置存储级别来禁用序列化。减少序列化可以降低CPU的工作负载，此时如果没有引入过多的GC消耗的话，可以提高系统的性能。

#### 1.4 任务启动开销
如果每秒钟启动的任务过多(比如超过50个/s)，此时将任务分发到从节点上的开销会是巨大的，很难达到亚秒级别的延迟。可以通过以下方式来减少这种消耗：
- **执行模式**，在Standalone或Mesos粗粒度模式下，启动Spark任务所需的时间比在Mesos细粒度模式下要短。更多细节可以查看在[Running on Mesos Guide](https://spark.apache.org/docs/latest/running-on-mesos.html)。

这些修改大约能减少批处理时间100ms，使得亚秒级别的批次大小成为可能。

### 2. 设置合适的批次间隔
对于一个运行在集群上的SparkStreaming程序来讲，要想保持稳定，系统一定要能尽快的将受到的数据处理掉。也就是说，数据的处理速度要能跟的上数据的生成速度。可以通过web UI查看批次处理时间是否小于批次间隔，来确定这一点。

由于Streaming计算的本质，对于一个运行在有固定资源的cluster上的应用程序来讲，batch interval对于其数据处理能力有很大的映像。例如，考虑早先的`WordCountNetwork`例子。对于一个确定的数据生成速率，系统能跟得上每2s(batch interval为2s)报告一次单词计数的频率，但可能无法跟得上每500ms报告一次的频率。因此需要谨慎的设置batch interval，确保能维持住数据的生成速度。

要找出应用的合适的批次间隔，可以先使用一个保守的间隔(如5-10s)在一个较低的数据速率上测试。要验证系统能否跟得上数据的生成速率，可以查看每个批任务中的延迟。如果延迟一直小于批间隔，那么系统就是稳定的。如果延迟在不断的增长，那就是系统无法跟上数据的生成速度。当有了稳定的配置后，可以尝试增大数据生成速率，或降低批处理时间，进一步进行测试。

### 3. 内存优化
在[Tuning Guide](https://spark.apache.org/docs/latest/tuning.html#memory-tuning)中详细讨论了如何对内存以及GC进行优化。强烈建议阅读一下。这一节，我们讨论下SparkStreaming语境下的一些特定的参数。
SparkStreaming程序锁需要的集群内存资源大小，严重依赖于所要执行的操作是何种类型。例如，如果要在过去10min的数据上执行window操作，那么集群就需要有足够的内存来容纳这10min的数据。再比如说想要在大量的key上使用`updateStateByKey`函数，也需要很多的内存。相反，如果仅仅是简单的map-filter-store操作，那么所需要的内存就会很少。
一般来讲，receiver接收到的数据会使用存储级别`StorageLevel.MEMORY_AND_DISK_SER_2`来存放，也就是内存中放不下的数据会溢写到磁盘上。落盘的过程会降低Streaming程序的执行速度，因此建议提供Streaming程序所需要的足够的内存。最好是在一边放大内存，一边观察。
内存优化的另一方面就是GC。对于想要低延迟的Streaming程序来讲，由于JVM的GC引起的大量的暂停是不希望看到的。
下面是几个可以对内存使用和GC开销进行优化的参数：

- **DStream的持久化级别**，如在序列化中提到的，数据的数据和RDD，默认是以序列化的形式持久保存的。这一举措既能降低内存占用，也能减少GC消耗。启动Kryo序列化，能够进一步降低序列化大小和内存占用。在此基础上，想要进一步减少内存占用，可以牺牲一部分的CPU资源，将数据进行压缩(具体可以查看配置选项`spark.rdd.compress`)
- **清除旧数据**，默认情况下，所有的输入数据，以及DStream转换过程中产生的持久化RDD都会被自动清理。SparkStreaming会根据使用的转换操作类型决定什么时候对对数据进行清理。例如，如果使用10min的窗口操作，SparkStreaming会保留大约过去10min的数据，丢弃更旧的数据。可以通过设定`streamingContext.remember`来获得数据更久的保存时间。
- **CMS垃圾回收器**，强烈建议使用CMS垃圾回收器，来保持GC相关的暂停始终较少。尽管concurrent GC会降低系统整体的吞吐量，但为了获得更平均的批处理时间，仍然建议使用。确保在Driver(提交程序时使用`--driver-java-options`命令行选项或在配置文件中配置`spark.driver.extraJavaOptions`)和Executor(使用配置`spark.executor.extraJavaOptions`)上都设置了CMS GC。
- **其他提示**，为进一步减少GC的消耗，下面是几个值得尝试的点：
  - 使用`OFF_HEAP`存储级别持久化RDD。更详细的讨论在[Spark Programming Guide](https://spark.apache.org/docs/latest/rdd-programming-guide.html#rdd-persistence)中
  - 使用带有更小堆内存的更多机器。这个会给每个JVM堆减小GC的压力

### 4. 需要记住的重要的几个点
1. 一个DStream会被关联到一个单独的receiver。为了达到并行读取的目的，需要创建多个DStream。receiver在Executor中运行，并占据一个core。要确保receiver占用了一部分core之后，仍有足够的core可供使用。参数`spark.cores.max`应该把receiver所占用的core的个数计算在内。receiver使用round robin方式分配到executor中。
2. 从数据源中接收数据，receiver会创建数据块(data block)。产生block的间隔由参数`spark.streaming.blockInterval`决定(单位是ms)。由此，在一个batch的时间间隔内，会产生N个block，其中`N=batchInterval/blockInterval`。这些block由当前Executor的BlockManager分发到其他Executor中的BlockManager中。在这之后，Driver上运行的Network Input Tracker会被通知到各个Block的位置，从而做进一步的处理。
3. 在一个batchInterval中所有的Block组成一个RDD，每个block都是这个RDD的一个partition，每个Partition都是Spark中的一个task。因此，如果blockInterval等于batchInterval，那么就只会产生一个partition，这种情况一般会在本地直接处理掉。
4. Block上执行的map操作直接在存放Block的Executor中执行(除了接收Block的receiver所在的Executor外，为防止意外发生，Block在其他Executor中还会有备份，所有的Executor中都会执行这个任务)，而不会去关心blockInterval，除非启用了non-local Scheduling。调高参数`spark.streaming.blockInterval`意味着更大的Block size。调高`spark.locality.wait`能够提高数据本地化的概率。为保证数据执行的本地化，需要平衡这两个参数
5. 也可以通过调用`inputDstream.repartition(n)`来主动对RDD进行分区(n个)，而不是使用配置参数batchInterval和blockInterval，从而提高并行度，尽管这样会有shuffle的代价。job由Driver的JobScheduler调度，同一时间只会有一个Job处于激活状态，因此剩余的Job会等待排队执行。
6. 如果有两个DStream，那么就会形成两个RDD，也就是会有两个job。这两个job都是在Driver的JobScheduler中调度，并且一次执行一个，即job要排队执行。要避免这一点，可以将两个DStream联合起来，从而形成一个单一的联合DStream。这个联合的DStream只会创建一个job。联合并不会影响RDD的分区。
7. 如果批处理时间要大于批次接收间隔，那么显然receiver的内存占用会不断增大，导致最终抛出异常(一般是`BlockNotFoundException`)。目前没有办法能够暂停receiver，不过使用配置项`spark.streaming.receiver.maxRate`可以限制接收者的接收速度。
