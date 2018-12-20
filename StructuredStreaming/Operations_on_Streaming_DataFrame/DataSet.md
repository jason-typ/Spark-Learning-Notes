判断一个DataSet/DataFrame是否是流式数据，使用`isStreaming`方法，返回布尔值
```
dataset.isStreaming
```
### Window Operation
下面是一个基于时间统计window内事件的例子
```
val spark = SparkSession
      .builder
      .appName("StructuredNetworkWordCountWindowed")
      .getOrCreate()

    import spark.implicits._

    // Create DataFrame representing the stream of input lines from connection to host:port
    val lines = spark.readStream
      .format("socket")
      .option("host", host)
      .option("port", port)
      .option("includeTimestamp", true)
      .load()

    // Split the lines into words, retaining timestamps
    val words = lines.as[(String, Timestamp)].flatMap(line =>
      line._1.split(" ").map(word => (word, line._2))
    ).toDF("word", "timestamp")
```
![](https://spark.apache.org/docs/latest/img/structured-streaming-window.png)

从12点开始，窗口大小为10分钟，每5分钟统计一次。对于12:07到来的一条数据，会被两个窗口统计到：12:00-12:10和12:05-12:15。

对于延迟收到的数据，在Window中会更新过去的对应的时间段。

![](https://spark.apache.org/docs/latest/img/structured-streaming-late-data.png)
比如在12:13收到了一条12:04的数据，这样需要去更新对应的时间段的result table。

### watermark
result结果保存在内存中，但流是无限增大的，随着时间推移，需要能把旧的数据从内存中移除。换句话说：不再更改这些数据。WaterMark作用就是这个：对于某个旧的聚合，由于系统不会再接收该聚合内的数据(收到了也会丢弃)，所以不会更新这个聚合，所以可以放心的把这个聚合从内存中移除。

```Scala
import spark.implicits._

val words = ... // streaming DataFrame of schema { timestamp: Timestamp, word: String }

// Group the data by window and word and compute the count of each group
val windowedCounts = words
    .withWatermark("timestamp", "10 minutes")
    .groupBy(
        window($"timestamp", "10 minutes", "5 minutes"),
        $"word")
    .count()
```
![](https://spark.apache.org/docs/latest/img/structured-streaming-watermark-update-mode.png)
在timestamp上定义watermark，约束延迟到达10min以上的数据会被丢弃(10分钟以前的聚合不会再更改)。图中的蓝线表示收到的事件的最大时间，Spark计算waterMark的方式就是这个最大时间减去设定的值(10min)。比如在12:15，在处理所有事件后，发现前面收到的事件中最大的时间是12:14，因此更新watermark为12:04。到了12:20，此时的watermark还没变，重复前面的操作，在这段时间内收到的事件的最大时间为12:21，更新之后的watermark为12:11。(改变watermark都是为了下一次trigger时设定的)

上面的是Update模式：收到一个late的数据，在处理后可以更改输出。但有些sink不支持这么干，此时需要用到append mode：在确定该时间段内的数据不再会被修改之后，再更新result table，并输出。

![](https://spark.apache.org/docs/latest/img/structured-streaming-watermark-append-mode.png)

比如在12:20trigger的时候，watermark还处于12:04，也就是说12:00-12:10时间段内的结果还可能会发生变化，因此这个时候不会产生输出。直到到了12：20trigger时，watermark已经被设置为12:11，这之前的数据都会被丢弃，也即是说12:00-12:10时间段内的结果不会再发生改变。因此才会输出结果。

### Join Option
Structured Streaming支持streaming DataFrame/DataSet与另一个static或Streaming DataFrame/DataSet执行join操作。

streaming join的结果一般都是逐渐增长的(stream在逐渐增长)，结果的类型与两个静态的DataFrame/DataSet执行join的结果类型相同。

##### 1. Stream-Static join
```
val staticDf = spark.read. ...
val streamingDf = spark.readStream. ...

streamingDf.join(staticDf, "type")          // inner equi-join with a static DF
streamingDf.join(staticDf, "type", "right_join")  // right outer join with a static DF
```
