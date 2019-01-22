Spark提供了两套API：low-level unstructured API 和 high-level Structured API。

# SparkStreaming与Kinesis的集成

### 整体结构
![Spark与Kinesis的集成](https://spark.apache.org/docs/latest/img/streaming-kinesis-arch.png)

SparkStreaming的DStream会调用KCL，在KCL中为每个shard启用一个读取线程，消费Shard中的数据。一个Shard对应一个读取线程，一个DStream中可以包含多个读取线程。因此，如果只启用了一个DStream，在这个DStream中就会创建Shard个数个线程来分别读取每个Shard。

一般会启动多个DStream，比如，有多少个Shard就启用多少个DStream，这样，每个DStream中都会有一个KCL，并只有一个线程，来消费一个Shard。KCL应用(Kinesis DStream)通过DynamoDB进行协同，把checkpoint保存在DynamoDB中，并且当某个DStream失效，其他的DStream也会检测到，并启动一个processor线程关联到这个Shard。不同的DStream通过KCL应用名称来确定协同关系。DStream的数量不应该超过Shard的数量，否则就是资源浪费。

### 代码编写
#### 1. 配置
引入包依赖
```sbt
"org.apache.spark" %% "spark-streaming-kinesis-asl" % sparkVersion
```

#### 2. 代码编写

```scala
val kinesisStream = KinesisInputDStream.builder
    .streamingContext(streamingContext)
    .endpointUrl([endpoint URL])
    .regionName([region name])
    .streamName([streamName])
    .initialPositionInStream([initial position])
    .checkpointAppName([Kinesis app name])
    .checkpointInterval([checkpoint interval])
    .storageLevel(StorageLevel.MEMORY_AND_DISK_2)
    .buildWithMessageHandler([message handler])
```
  - streamingContext，其中需要包含应用的名称(作为Kinesis的消费者)
  - `[Kinesis app name]`，应用的名称，KCL会使用DynamoDB保存当前Shard消费到的Record对应的序列号，称为checkpoint，在重启时可以知道从流的何处继续读取数据
    - 在给定的账户和区域下，应用的名称必须唯一，因为这个名字会作为DynamoDB表的名字
    - 如果DynamoDB中存在以该应用名称命名的表，但里面的checkpoint信息错误，此时可能会有错误发生
  - `[Kinesis stream name]`，Kinesis Stream的名字
  - `[endpoint URL]`，[Kinesis的终端节点地址](https://docs.aws.amazon.com/zh_cn/general/latest/gr/rande.html#ak_region)
  - `[region name]`，[区域](https://docs.aws.amazon.com/zh_cn/general/latest/gr/rande.html#ak_region)
  - `[checkpoint interval]`，KCL保存checkpoint到DynamoDB的时间间隔。这个间隔与SparkStreaming采集多长时间间隔内的数据作为一个batch来处理里面的间隔不是一个，但可以设置为相同的值
  - `[initial position]`，可以是`InitialPositionInStream.TRIM_HORIZON`或者 `InitialPositionInStream.LATEST`，表示没有检查点时从哪个位置开始读取
  - `[message handler]`，处理函数，输入是Kinesis中的记录，输出自己定义
