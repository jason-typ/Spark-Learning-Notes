## SparkStreaming中的checkpoint


A streaming application must operate 24/7 and hence must be resilient to failures unrelated to the application logic (e.g., system failures, JVM crashes, etc.). For this to be possible, Spark Streaming needs to checkpoint enough information to a fault- tolerant storage system such that it can recover from failures. There are two types of data that are checkpointed.

Metadata checkpointing - Saving of the information defining the streaming computation to fault-tolerant storage like HDFS. This is used to recover from failure of the node running the driver of the streaming application (discussed in detail later). Metadata includes:
Configuration - The configuration that was used to create the streaming application.
DStream operations - The set of DStream operations that define the streaming application.
Incomplete batches - Batches whose jobs are queued but have not completed yet.
Data checkpointing - Saving of the generated RDDs to reliable storage. This is necessary in some stateful transformations that combine data across multiple batches. In such transformations, the generated RDDs depend on RDDs of previous batches, which causes the length of the dependency chain to keep increasing with time. To avoid such unbounded increases in recovery time (proportional to dependency chain), intermediate RDDs of stateful transformations are periodically checkpointed to reliable storage (e.g. HDFS) to cut off the dependency chains.
To summarize, metadata checkpointing is primarily needed for recovery from driver failures, whereas data or RDD checkpointing is necessary even for basic functioning if stateful transformations are used.

### 为什么需要checkpoint
Spark的失败恢复依赖于RDD的血缘关系：只要最开始的数据没有丢失，任何一个丢失的RDD都可以根据血缘关系重新计算得到。如果血缘关系过长，会导致重新计算的代价过大，因此如果此时将中间的某个RDD保存到文件系统中，会提高从失败中恢复的效率。另外对于Streaming程序来说，通常无法将所有数据replay一遍，因此如果提供了状态服务，或者希望能从意外中恢复driver，也都需要checkpoint机制。

### SparkStreaming中checkpoint的执行过程

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

在方法`doCheckpoint`中，会根据时间判断是否需要执行checkpoint。

另外，`JobGenerator`类中还有一个定时器，用于定期产生事件：
```scala
private val timer = new RecurringTimer(clock, ssc.graph.batchDuration.milliseconds,
  longTime => eventLoop.post(GenerateJobs(new Time(longTime))), "JobGenerator")
```

这样，整个checkpoint流程就比较清楚了：在`JobGenerator`中有一个定时器，根据最开始定义的batchDuration，决定多久生成一个batchJob，放到队列中等待执行；batch job执行结束后，产生一个`DoCheckpoint`事件放入队列，等待执行；调用方法`doCheckpoint`，执行checkpoint操作。即每执行一个batch job，都会根据时间判断是否需要执行一次checkpoint，需要的话就执行。

### checkpoint使用
