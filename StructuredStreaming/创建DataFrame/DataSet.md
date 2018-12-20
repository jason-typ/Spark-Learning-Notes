Streaming DataFrame使用`DataStreamReader`接口(这个接口直接由`SparkSession.readStream`创建并返回)创建。这里可以对Source进行详细的配置，如：

```Scala
val kinesis = spark
      .readStream
      .format("kinesis")
      .option("streamName", "tangtest")
      .option("endpointUrl", "kinesis.cn-north-1.amazonaws.com.cn")
      .option("awsAccessKeyId", "keyId")
      .option("awsSecretKey", "key")
      .option("startingposition", "TRIM_HORIZON")
      .load()
```

### InputSource
Structured Streaming API提供了一些内置的输入源的支持，包括(目前不支持Kinesis)：

- File Source。某个文件夹下的所有文件作为数据源
- Kafka
- Socket(测试用)。从socket中读取文本数据
- Rate(测试用)

以socket作为数据源的示例如下：

```Scala
val spark: SparkSession = ...

// Read text from socket
val socketDF = spark
  .readStream
  .format("socket")
  .option("host", "localhost")
  .option("port", 9999)
  .load()
```
得到的是DataFrame(每一行的类型都是Row)。对于map这类操作，需要知道具体的类型，可以把DataFrame转换成DataSet后再进行操作。
