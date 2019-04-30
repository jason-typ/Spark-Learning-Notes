[参考](https://databricks.com/blog/2015/01/15/improved-driver-fault-tolerance-and-zero-data-loss-in-spark-streaming.html)

### 1. 什么是SparkStreaming的WAL
实时流程序需要7*24小时不间断的运行，这就要求系统具有从失败中恢复的能力。与其他系统一样，Spark的WAL是在1.2版本中引入的，为防止系统在从失败中恢复的过程中引起的数据丢失的问题。

### 2. SparkStreaming为什么需要WAL
前面介绍过SparkStreaming的Checkpoint机制，能够让SparkStreaming从失败中恢复。Checkpoint使得应用程序周期性的将自身的状态保存到可靠的文件系统上，从而在重启时恢复之前的状态。Checkpoint另一个功能就是斩断RDD之间的依赖链，防止这个依赖链过长，因此在Statefule的程序中特别好用。

但Checkpoint机制无法解决一个问题：数据的丢失。考虑一个流式数据源，如Kafka，receiver从Kafka中读取数据并保存在内存中，在对数据进行处理前，Driver挂掉了(所有的Executors也会终止)，或某个Executor自己挂掉了，那么缓存在Executor中的数据就会丢失。为了解决这种数据丢失的问题，Spark在1.2版本中引入了WAL机制。

对于像HDFS、S3这类可靠的数据源来说，其实并不需要WAL，Checkpoint机制完全可以应对从失败中恢复。因为对于HDFS这里数据来源，数据读取完全可以replay一遍，只要RDD的信息没有丢失，可以重新从文件系统中重新读取。而对于Kafka这类数据源来说，创建RDD的数据源其实是存放在内存中的，RDD的信息也是记录的内存中的数据。因此，只有Kafka这类数据源(从中读取数据并放在内存中以待后续处理)才需要WAL机制。

### 3. WAL机制
WAL用在数据库和文件系统中，用来保证对数据的操作的持久性。主要原理是将对数据的操作先持久化，然后再执行这些操作。假如在执行这些操作的过程中失败了，那么系统在重启后可以replay这些已持久化的operations。

Spark的WAL机制也是同样，我们说HDFS数据源不需要WAL，那么就对Kafka这类数据源，在接收到数据后，首先落盘存储，保证数据源的可靠，如此一来就不会出现数据丢失的问题了。以Kafka为例，加入WAL机制后大致流程如下图：

![wal-kafka](https://databricks.com/wp-content/uploads/2015/03/Screen-Shot-2015-03-29-at-10.11.42-PM.png)

1. 运行在Executor中的Receiver持续不断的接收来自Kafka的数据
2. Receiver接收到数据后，不仅仅存放在内存中，还会将这部分数据写入到磁盘上。在写入磁盘上之后，通知Kafka已接收到数据，更新offset
3. Driver对这部分接收到的数据产生新的batch job

因为只有在将数据落盘后，才会更新Kafka中的offset，那就确保了这部分数据一定会被处理到。在这种模式下，SparkStreaming可以保证不再会有数据丢失的情况。即SparkStreaming通过WAL机制提供了At Least Once的处理能力。

### 4. 如何使用WAL
WAL是一个内部机制，使用时只需要做相应的配置：
1. 设置Checkpoint目录。这个目录会被用来保存Checkpoint数据，以及WAL数据。
2. 打开配置。设置`spark.streaming.receiver.writeAheadLog.enable`为`true`

### 5. WAL的影响
由于所有接收到的数据都需要首先保存在可靠的文件系统中，因此WAL对Streaming的吞吐量影响很大，对整个系统的性能影响很大。如果可以接受数据的少量丢失，那么建议不要开启WAL。

另外，WAL可能会造成数据的重复处理。WAL+Checkpoint解决的是从失败中恢复以及数据的丢失的问题：加入一个job在执行过程中失败了，重启后根据Checkpoint重建StreamingContext以及这些incomplete的job。job从WAL中读取数据。

但是如果这个incomplete job已经写了一部分数据到Sink中了，重新执行这个job时，之前失败的job中已经写入Sink中的那部分数据就会成为重复数据。

### 6. WAL的执行流程
WAL在SparkStreaming执行过程中是如何使用的，如下图。

![wal_process](https://databricks.com/wp-content/uploads/2015/01/blog-ha-52.png)

整个系统启动成功后，从Receiver开始接收数据开始，流程大致如下：
1. 接收数据。Receiver将接收到的数据组成Data Block，存放在内存中。如果启用了WAL，那么数据还会被存放到可靠文件系统中一份
2. 运行在Executor上的TrackerSupervisor将收到的数据的元数据通知给Driver端的ReceiverTracker(绿色箭头)。这部分元数据也会写入到Log中。元数据包括：
    - 数据在Executor内存中的位置
    - 如果启用了WAL，还包括数据在WAL文件中的offset
3. 数据处理。StreamingContext中的JobGenerator根据Block Data的元数据信息创建RDD并生成在RDD上待执行的job。Job负责处理存放在Executor内存中的数据
4. Checkpoint。最后，周期性的将执行状态通过Checkpoint机制保存在内存中，以便从失败中恢复

一个Driver从失败中恢复的流程如下：

![Driver-restart-with-WAL](https://databricks.com/wp-content/uploads/2015/01/blog-ha-4.jpg)

1. 根据Checkpoint重建StreamingContext(橙色箭头)
    读取Checkpoint数据，重建StreamingContext，启动Executors
2. 恢复Block数据的metadata(绿色箭头)。
    要继续处理数据，首先需要获得数据的元数据信息(数据保存在哪儿)
3. 重新生成未完成的job(红色箭头)
    恢复了元数据后，就可以根据元数据以及已经完成的job的信息，生成还没有完成的job
4. 执行Job
    Job分发到Executor后，Executor从WAL文件中读取数据，开始执行失败前没有完成的job
5. 继续正常的流程
    Receiver继续从Source中读取数据，继续下面的数据处理
