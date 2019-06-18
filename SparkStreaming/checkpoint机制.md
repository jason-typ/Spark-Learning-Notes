### checkpoint与其作用
为保证一个Streaming程序能够7*24小时不间断的正常运行，应用程序必须对外界的一些失败情况(如，系统异常、JVM崩溃等)具有抗干扰的能力。CheckPoint机制就是为了应对这种情况，让系统能够从失败中恢复。

可以先考虑一下，如果一个Streaming程序由于机器原因崩溃了，如何才能从原先的执行状态继续进行下去。Spark中有两种节点：Driver和Executor。Driver负责任务的分解以及分发、监控，Executor负责任务的实际计算。要保证系统能从失败的位置恢复，就必须能够分别应对这两种不同节点失败的情形。而这两种节点性质的不同，也就决定了checkpoint对他们保存的状态的不同。两种需要checkpoint的数据分别为：
- **Metadata**
  应对Driver节点失效。Driver节点控制整个Job的执行流程，而不是Job的执行，因此需要保存的是执行的元数据信息，包括：
  - **Configuration**：创建应用程序的配置信息
  - **DStream操作**：Streaming程序就是操作DStream，这一部分就是保存Application需要做什么
  - **Incomplete Batches**：SparkStreaming是基于微批的，一个Streaming Interval中会被分为多个batch，这部分就是记录哪些batch还没有完成，重启后就需要重新执行哪些batch
- **Data**
  应对Executor节点失效。Executor负责task的执行，需要保存的是执行生成的RDD数据。如果Executor在机器崩溃时处于task执行的中间过程时，此时task对应的中间RDD的数据也会丢失。这种RDD丢失没什么影响，最多将整个batch重新计算一遍。但是如果是涉及到状态的管理，后一个batch的状态需要依赖于前一个batch的状态进行修改时，用于管理状态的RDD就不断形成向前一个状态RDD的依赖关系。此时如果机器崩溃，那么状态RDD也就丢失了。整个状态管理就需要从头开始重新执行一遍。更可怕的是，对Streaming程序来说，它的source经常是不可replay的(可能被删除了)。此时checkpoint机制就会特别有用，所以在有状态管理的Streaming程序中，一定要使用checkpoint

从Checkpoint的作用中，也能看出来什么时候需要使用Checkpoint：
- 状态管理
  当业务代码中涉及到多个Streaming Interval之间的依赖关系，或者说使用了`updateStateByKey`这类函数时
- Driver失败恢复
  需要Driver能从失败中恢复

### checkpoint与cache
cache与checkpoint好像有点相似，但功能是完全不同的。
- cache
  缓存的目的是为了避免重复计算某个RDD。Spark中每一次在一个RDD上调用Action操作，都会触发RDD以及其整个依赖的重新计算，消耗太大。因此才会需要将这个会反复用到的RDD标记为需要持久化，从而避免重复计算。另外，从缓存的位置来看，缓存可以放在内存或磁盘中，在整个机器崩溃进行恢复时，基本无法做到缓存数据的恢复。
- checkpoint
  checkpoint的目的是为了失败恢复。为了保证checkpoint信息在机器down掉后还能被获取到，checkpoint一般都是需要放在可靠的位置的，如HDFS、S3等

### checkpoint如何使用
以下面代码为例：
```Scala
def createContext(option: DimeJobOption): StreamingContext = {
  logger.error(s"Creating new context for ${option.appName}")
  val conf = new SparkConf()
    .setAppName(option.appName)
    .registerKryoClasses(
      Array(
        classOf[Record], classOf[SessionState], classOf[Session], classOf[User], classOf[DimeJobOption]))
    .setMaster("local[3]")

  val ssc = new StreamingContext(conf, Seconds(option.streamingBatchInterval))
  // send a heart beat message to Pandora to monitor Spark Streaming's health.
  ssc.addStreamingListener(new StreamingMonitor(option.appName))
  ssc.checkpoint(option.checkpointDirectory)

  val pageIDMapBroadcast = ssc.sparkContext.broadcast(pageIDMap.getMap(ssc.sparkContext))
  val stream = KinesisInputDStream.builder
    .streamingContext(ssc)
    .regionName(option.region)
    .streamName(option.stream)
    .endpointUrl(option.endPointURL)
    .checkpointAppName(s"dime-${option.appName}-kinesis")
    .checkpointInterval(Seconds(option.checkpointInterval))
    .initialPosition(KinesisInitialPositions.fromKinesisInitialPosition(InitialPositionInStream.LATEST))
    // get first 38 bytes as id
    .buildWithMessageHandler {
    rec =>
      val r = Record.parseFrom(rec.getData().array())
      val id = rec.getSequenceNumber.substring(38).toLong
      (r, id)
  }
    .filter {
      case (r, _) => option.namespaces.contains(r.namespace)
      case _ => false
    }
    .checkpoint(Seconds(5 * option.streamingBatchInterval))

  val stateSpec =
    StateSpec
      .function {
        (key: (Int, String, String), value: Option[(Record, Long)], state: State[SessionState]) =>
          updateSessions(key, value, state, option)
      }
      // if after `time` out, no events of existing keys arrive, the `updateSessions` function
      // will be invoked for the last time and then remove these key.
      .timeout(Seconds(60))

  stream.map {
    ...
  }
    .mapWithState(stateSpec)
    .flatMap(s => s)
    .foreachRDD { (rdd) =>
      rdd.foreachPartition { (states) =>
        // write SessionState into DynamoDB's fact and user table
        DataWriter.writeSessionState(option, states)
      }
    }

  ssc
}

def run(option: DimeJobOption) {
  val ssc = StreamingContext.getOrCreate(option.checkpointDirectory,
    () => createContext(option))

  logger.info(s"event=start_running_streaming_task")
  ssc.start()
  ssc.awaitTermination()
}
```
Checkpoint的使用方式是固定的：使用`StreamingContext。getOrCreate`方法，创建一个新的，或从指定的目录中读取旧的。
- 假如从指定目录找到了Checkpoint缓存，根据Checkpoint目录内容加载StreamingContext
- 假如指定目录没有Checkpoint，那么就使用提供的函数创建一个新的StreamingContext

### CheckPoint使用注意事项
Checkpoint的使用有两个坑：
1. 处理逻辑必须写在functionToCreateContext函数中，即上面的`createContext`函数，而不能直接写在main方法，或上面例子中的`run`方法中，一定是要在`createContext`函数中
  原因不是很清楚，但是如果不这样(写在这个函数中)，程序第一次启动时，Checkpoint文件夹不存在，程序可以正常启动。但是在程序退出，重新启动时，会发现读取Checkpoint错误，启动失败。提示原因如：
  ```
  [[org.apache.spark.SparkException: org.apache.spark.streaming.dstream.FlatMappedDStream@7f2269ec has not been initialized
        at org.apache.spark.streaming.dstream.DStream.isTimeValid(DStream.scala:313) ~[spark-streaming_2.11-2.3.0.jar:2.3.0]
        ...
  ```
2. 在网上看到一个问题，在修改程序，重新提交后，发现报错`ClassNotFoundException`，原因是Checkpoint的元数据会记录jar的序列化的二进制文件。由于代码重新编译，新的序列化jar文件在Checkpoint记录中并不存在，所以产生了这个报错。解决办法是删除checkpoint开头的文件即可，不影响数据本身的Checkpoint：```hdfs dfs -rm /spark/***/checkpoint/checkpoint*```
  具体没有试过，先记录下来

3. 除了定时自动的完成Checkpoint工作外，还可以在RDD上主动的调用`checkpoint`函数，将一个RDD标记为需要执行Checkpoint。这种情况需要注意，调用`checkpoint`函数一定是要在对这个RDD执行任何操作之前完成。另外，强烈建议将这个需要执行Checkpoint的RDD缓存起来，否则在执行Checkpoint的时候，又会对这个RDD重新计算一遍，浪费计算资源。执行这个操作后，所有对这个RDD的父RDD的引用都会被删除

### CheckPoint的调用过程
从`JobGenerator`开始说起，`JobGenerator`用于定时生成batch job并提交。`JobGenerator`其实就是一个Akka并发模型Actor的实现：
```scala
eventLoop = new EventLoop[JobGeneratorEvent]("JobGenerator") {
  override protected def onReceive(event: JobGeneratorEvent): Unit = processEvent(event)

  override protected def onError(e: Throwable): Unit = {
    jobScheduler.reportError("Error in job generator", e)
  }
}
```
`onReceive`方法定义了在收到event时如何处理，`processEvent`的实现如：
```scala
private def processEvent(event: JobGeneratorEvent) {
  logDebug("Got event " + event)
  event match {
    case GenerateJobs(time) => generateJobs(time)
    case ClearMetadata(time) => clearMetadata(time)
    case DoCheckpoint(time, clearCheckpointDataLater) =>
      doCheckpoint(time, clearCheckpointDataLater)
    case ClearCheckpointData(time) => clearCheckpointData(time)
  }
}
```
一共四种类型事件：其中`GenerateJobs`是生成batch job，`DoCheckpoint`是执行checkpoint操作。

在方法`doCheckpoint`中，会根据时间判断是否需要执行checkpoint:

```Scala
if (shouldCheckpoint && (time - graph.zeroTime).isMultipleOf(ssc.checkpointDuration)) {
  logInfo("Checkpointing graph for time " + time)
  ssc.graph.updateCheckpointData(time)
  checkpointWriter.write(new Checkpoint(ssc, time), clearCheckpointDataLater)
} else if (clearCheckpointDataLater) {
  markBatchFullyProcessed(time)
}
```
其中`shouldCheckpoint`根据是否配置了CheckPoint目录以及时间间隔来判定。另外，方法`isMultipleOf`判断的是是否是整数倍。所以，设置CheckPoint时间间隔需要是batch interval的整数倍才可以。假如batch interval设为10s，CheckPoint interval设为7s，那么恐怕每70s才会执行一次CheckPoint。

另外，`JobGenerator`类中还有一个定时器，用于定期产生事件：
```scala
private val timer = new RecurringTimer(clock, ssc.graph.batchDuration.milliseconds,
  longTime => eventLoop.post(GenerateJobs(new Time(longTime))), "JobGenerator")
```

这样，整个checkpoint流程就比较清楚了：在`JobGenerator`中有一个定时器，根据最开始定义的batchDuration，决定多久生成一个batchJob，放到队列中等待执行；batch job执行结束后，产生一个`DoCheckpoint`事件放入队列，等待执行；调用方法`doCheckpoint`，执行checkpoint操作。即每执行一个batch job，都会根据时间判断是否需要执行一次checkpoint，需要的话就执行。

### CheckPoint执行过程
上面了解了SparkStreaming中是如何调用到执行CheckPoint的：每次生成一个batchjob后，都会提交一个doCheckPoint操作。通过判断当前时间是否需要执行CheckPoint来最终决定是否要执行CheckPoint操作。下面了解一下具体的CheckPoint操作是咋么执行的，也就是`doCheckpoint`这个函数。

在write函数中，定义了一个runnable对象`CheckpointWriteHandler`。在`run`方法中，写的过程有以下几个步骤：
1. 将新的Checkpoint写到一个临时文件夹下
2. 每个时刻都有一个当前Checkpoint文件A，以及上一时刻的Checkpoint文件的备份B。在Checkpoint一个新的文件时，首先删除B，然后把A重命名为B
3. 将Checkpoint临时文件夹重命名为正式的Checkpoint文件
4. 删除旧的、多余的checkpoin文件(这里不是很明白)
5. 执行`onComplete`回调

### Checkpoint与广播变量和累加器
SparkStreaming中Checkpoint与广播变量和累加器无法一起工作，可以算是Spark的一个bug，具体可以见 [Jira5206](https://issues.apache.org/jira/browse/SPARK-5206)。但是算是给出了一个workaround的方法：将广播变量或累加器做成一个懒加载的单例对象。具体例子可以查看 [GitHub commit](https://github.com/apache/spark/pull/10385)或Spark源码中的`RecoverablenetworkWordCount`类。这里不具体介绍。

但是在我按照如上方法将广播变量放在一个懒加载对象中时，还是出错，原因是：
```
Task not serializable
```

这个的原因(不是完全明白)是因为Spark本质上是一个分布式的计算框架，执行过程简单来说：把计算operation序列化后发送到各个节点上，在各个节点上反序列化完成计算任务。但是在序列化task的时候，发现无法完成序列化的工作，从而报了这个错误。解决方法是在Scala中，最好都使用函数，而不是方法定义。

这个解决之后，又遇到了另外的问题：
```
java.lang.ClassCastException: org.apache.spark.util.SerializableConfiguration cannot be cast to scala.collection.MapLike
```
最后由于要使用的广播变量很小，选择先不广播了。。

### checkpoint到S3上遇到不一致问题
为保证集群能无状态重启，将Streaming任务的Checkpoint目录设置为s3。程序运行一段时间后基本上都会出现Checkpoint失败的问题：
```
"Checkpoint RDD has a different number of partitions from original RDD. Original "
```
在`ReliableCheckpointRdd#writeRDDToCheckpointDirectory`中，会首先将Checkpoint数据oldRDD写到文件系统中，然后立即reload这部分数据作为newRDD。然后会使用相同的partitioner判断newRDD与oldRDD是否具有相同的parititon。两者不一致时，会抛出这个异常。

由于S3具有最终一致性，写下去后立马读取确实会有可能出现两者不一致的问题。解决方法是开启EMR的consistent view，或者要么自动重启Streaming任务就好。

### checkpoint与exactly once
