[StructuredStreaming](https://zhuanlan.zhihu.com/p/51883927)

Spark Streaming的不足：
1. 使用processing time而不是event time。processing time是指数据到达Spark被处理的时间，event time表示事件的发生时间，是事件本身的属性。我们知道Spark Streaming是基于DStream的micro-batch模式，这种模式下，批次的切割是基于processing time，这样就导致使用event time很困难。
2. 批/流API的不统一。开发人员需要为批处理和流处理分别写一套对应的代码。
3. reasoning about end-to-end guarantee

## 编程模型
Structured Streaming的关键点是将数据流当做一个不断增长的表。这样，流处理模型与批处理模型就很相似，Spark在这个可以无限增长的表上执行增量查询。

### 基本概念

将流式数据当做一张可以无限增长的表，每条新的数据都作为新的一行添加到表的最后。Structured Streaming编程模型的抽象是DataSet/DataStream。

![Data Stream as an unbounded table](https://spark.apache.org/docs/latest/img/structured-streaming-stream-as-a-table.png)

对输入数据(即Unbounded Table)的查询都会产生一个Result Table。每个触发间隔(比如1s)，新的数据会被追加到表中，并更新result table。只要result table更新了，我们就可以选择Result Table中的内容输出。

![Programming Model](https://spark.apache.org/docs/latest/img/structured-streaming-model.png)

Result表示内存中的计算结构，Output表示被写到外部存储设备中的部分。输出有三种模式，并且每种模式对query语句有所要求：
1. 全量输出。更新过的Result Table全部写入到外部存储设备中。
2. 追加输出模式。只有(自上一次trigger后)Result Table中新增加的行才会输出。这种模式要求Result Table中已经存在的row不能发生改动
3. 修改输出模式。(自上一次trigger后)Result Table中发生修改的部分会输出到外部存储。

### Quick Example
从socket中读取数据，并对读取到的数据进行单词统计。代码如下。
```Scala
import org.apache.spark.sql.SparkSession

object SparkLearning {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession
      .builder
      .master("local[2]")
      .appName("StructuredNetworkWordCount")
      .getOrCreate()
    import spark.implicits._

    // Create DataFrame representing the stream of input lines from connection to localhost:9999
    val lines = spark.readStream
      .format("socket")
      .option("host", "localhost")
      .option("port", 9999)
      .load()
    // Split the lines into words
    val words = lines.as[String].flatMap(_.split(" "))
    // Generate running word count
    val wordCounts = words.groupBy("value").count()
    // Start running the query that prints the running counts to the console
    val output = wordCounts.writeStream
      .outputMode("complete")
      .format("console")
      .start
    output.awaitTermination()
  }
}
```
首先需要创建一个`SparkSession`，这是整个Structured Streaming程序的开始。然后创建一个DataFrame，代表接收到的数据，表示编程模型中那个无限增长的表。到这里并没有开始接收数据。然后对数据进行计算，得到词频统计wordCounts。到这里，已经定义好了对流数据的query部分(没有实际开始执行)。最后剩下的就是定义输出，并真正开始处理流数据。下图表示上面的执行过程。

![model of quick example](https://spark.apache.org/docs/latest/img/structured-streaming-example-model.png)

所以在Structured Streaming中，共有三部分需要定义：input、query和output。上面的例子中，`lines`就是input，格式为`socket`。result是query的执行结果，query就是上面的计算过程，最终的wordCounts就是结果。最后是输出部分的定义。

Spark会持续的检查更新的数据，并将query应用到input table中新增的数据上，有时候还需要和之前的静态数据一起组合成结果。Spark中并不会保存整张表，input数据处理完后会被丢掉，只会保存更改result table所需要的数据。
