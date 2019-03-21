InputDStream代表输入到Spark中的数据流。每个InputDStream(除了文件流)都会与一个Receiver对象关联，这个Receiver对象负责从源接收数据，并将数据存放到内存中。

SparkStreaming提供了两种内置的源：
- **基本的源**，表示可以直接通过`StreamingContext`API使用的源，如文件、socket。
- **高级的源**，如Kafka、Flume、Kinesis等，这些源需要添加各自的依赖包来使用

Spark支持从多个源中并行的接收数据，只需要创建多个InputDStream即可。每个InputDStream都会自动的创建各自的Receiver负责接收来自各自源的数据。但要注意的是，一个Receiver会单独的占据一个core来接收数据，在Executor中处理数据也需要占用至少一个core，如果需要提高并行度启用多个task就需要多个core，因此一定要给Spark程序分配足够的core的数据。

注意的点：
- 本地运行SparkStreaming中，需要提供至少两个线程，如`setMaster("local[*]")`表示有多少个核就分配几个线程，`local[2]`表示分配两个线程。而`local`或`local[1]`都表示只分配一个线程，那么这个唯一的线程就会用来运行Receiver，从而就没有处理数据的线程了
- Cluster模式下，Receiver会选择一个Executor，在其中作为一个线程保持运行。Receiver占用一个单独的core
- 同理，在Cluster上运行也需要分配足够的core(Receiver启动在Executor中以线程的方式启动，如果Executor中所有的core都被Receiver占用了，自然就没有core用来运行其余的数据处理task了)。使用`num-executors`和`executor-cores`来配置

### 基本数据源

#### 文件源
从任何与HDFS API兼容的文件系统中读取文件都可以使用
```
StreamingContext.fileStream[KeyClass, ValueClass, InputFormatClass](dataDirectory)
```

读取文件不需要运行一个Receiver，因此不需要为读取文件数据分配core。如果是读取文本文件，最简单的方法是直接调用
```
StreamingContext.textFileStream(dataDirectory)
```

##### 1. 目录是如何被监控的
SparkStreaming会监控给定的目录，并处理在该目录下创建的任意文件。



Spark Streaming will monitor the directory dataDirectory and process any files created in that directory.

A simple directory can be monitored, such as "hdfs://namenode:8040/logs/". All files directly under such a path will be processed as they are discovered.
A POSIX glob pattern can be supplied, such as "hdfs://namenode:8040/logs/2017/*". Here, the DStream will consist of all files in the directories matching the pattern. That is: it is a pattern of directories, not of files in directories.
All files must be in the same data format.
A file is considered part of a time period based on its modification time, not its creation time.
Once processed, changes to a file within the current window will not cause the file to be reread. That is: updates are ignored.
The more files under a directory, the longer it will take to scan for changes — even if no files have been modified.
If a wildcard is used to identify directories, such as "hdfs://namenode:8040/logs/2016-*", renaming an entire directory to match the path will add the directory to the list of monitored directories. Only the files in the directory whose modification time is within the current window will be included in the stream.
Calling FileSystem.setTimes() to fix the timestamp is a way to have the file picked up in a later window, even if its contents have not changed.

### 接收器的可靠性
data source根据其可靠性可以分为两类。像Kafka和Flume这种，能够对收到的数据进行确认。因此，通过在Spark收到数据后进行确认，能够确保数据不会因为任何失败而丢失。这样就分为两种Receiver：
- **可靠接收器**，可靠接收器会在收到数据，并将数据以多个副本的形式存放在Spark内存中后，向数据源发送确认消息
- **不可靠接收器**，不可靠接收器不会像数据源发送确认消息。

关于如何配置一个可靠的接收器在[Custom Receiver Guide](https://spark.apache.org/docs/latest/streaming-custom-receivers.html)中介绍。
