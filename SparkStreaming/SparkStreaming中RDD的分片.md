# SparkStreaming中RDD的分片
SparkStreaming是微批计算：一个Interval中间接收到的数据作为一个RDD，从而与离线部分的处理方式相同。这个RDD中会有n个分区，n的计算方式是`batch interval / block interval`。block interval由配置`spark.streaming.blockInterval`指定，默认是200ms。batch interval是批处理间隔，在代码中指定。假如指定批处理间隔为2s，block interval使用默认的200ms，receiver负责从网络上接收数据，每200ms会将数据组成一个block后存放到Spark的内存中，作为RDD的一个partition，并且2s内接收到的数据都属于这个RDD。也就是说这个RDD中会有10个partition，每个partition包含200ms的数据。10个partition也就意味着10个task。

以上是查到的资料([链接1](https://stackoverflow.com/questions/32989171/rdd-partitioning-in-spark-streaming)，[链接2](https://forums.databricks.com/questions/778/how-many-partitions-does-spark-streaming-create-pe.html) )，但是在实际使用时发现，分区数量如果没有显式的指定，在执行时会根据给定的资源多少来分。比如给定了2个Executor，每个Executor2个Core，那么运行时就会分为4个Partition，也就是4个Task。也就是说分区数量会根据你给我多少资源来进行调整。
