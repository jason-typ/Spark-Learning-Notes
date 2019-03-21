Amazon Kinesis是大量流式数据实时处理的全托管服务。Kinesis receiver使用Amazon提供的KCL创建一个输入的DStream。KCL构建在Apache2.0许可之上，提供负载均衡、容错以及checkpoint机制。这里介绍如何配置SparkStreaming接收来自Kinesis的数据。

### 配置SparkStreaming程序
#### 1. 添加依赖
```
groupId = org.apache.spark
artifactId = spark-streaming-kinesis-asl_2.11
version = 2.4.0
```
#### 2. 代码编写
在Streaming程序中，引入`KinesisInputDStream`，并创建存放字节数组的输入`DStream`：



![SparkStreaming_Kinesis_Architecture](images/2019/02/sparkstreaming-kinesis-architecture.png)
需要注意的几点：
- Kinesis每个分区中的数据按顺序进行处理，并能提供at-least once的保证
- 一个Kinesis Stream可以被多个程序读取，Kinesis会将程序关联的分区以及当前的checkpoint保存在DynamoDB中
- Kinesis的一个分区同一时间只能被一个DStream处理
- 一个DStream可以通过创建多个`KinesisRecordProcessor`线程读取KinesisStream的多个分区
- 多个运行在不同线程/实例中的DStream可以读取同一个Kinesis Stream
- 由于每个DStream至少会创建一个`KinesisRecordProcessor`线程来处理一个单独的分区(shrd)，因此永远不需要创建个数多于Kinesis流分区数目的DStream
- 水平扩展可以通过增加/移除DStream来达到(在单独一个线程中，或衡阔多个线程/实例)。但如上所说，最大不超过分区数
- Kinesis DStream会在所有的DStreams中进行负载均衡，即使横跨多个线程/实例
- 即使由于负载变化而重新分区时，Kinesis DStream也会维持负载均衡
- 作为最佳实践，还是尽量避免动态的重新分区
- 每个Kinesis input DStream独立的维护它自己的checkpoint信息。可以查看Kinesis Checkpoint一节获取更多信息
- Kinesis stream的分区数与Spark创建的RDD的分区数之间没有关系，他们有不同的、独立的分区策略
### 运行实例
### Record De-aggregation
### Kinesis Checkpointing
- 每个Kinesis input DStream会周期性的把流当前的位置保存在DynamoDB的表中。这个可以让系统从失败中恢复，并且继续从上次处理的位置接着处理。
- 过滤频繁的保存checkpoint会引起AWS checkpoint存储层负载过重，从而导致AWS开始节流。提供的例子中使用一种随机避退重试机制来解决这个问题。
- 如果在input DStream启动时没有checkpoint村子，那么要么从Stream的最早一条记录(新的API推荐使用`KinesisInitlaPositions.Latest`)或最新的一条记录开始(`KinesisInitialPositions.Latest`)
- `KinesisInitialPositions.Latest`会导致在没有input DStream运行时存放到Kinesis中的数据丢失；`KinesisInitlaPositions.Latest`可能会导致数据的重复处理。
