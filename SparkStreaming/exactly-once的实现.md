### 1. exactly-once
exactly-once指的是对每一条数据，处理且仅处理一遍。相对应的还有：
- **at-least-once**:每条数据至少处理一次
- **at-most-once**:每条数据最多处理一次

要实现exactly-once这一目标，需要整个系统一起完成，不可能通过一个系统中某一个组件提供这一特性。以Spark来说，整体可以分为三个部分：数据源-数据处理-数据输出。需要这三个部分协调工作来共同实现exactly-once。

### 2. SparkStreaming的支持
SparkStreaming的处理过程本身是exactly-once的，而且对于上游数据源的支持做的很好。但对于下游Sink的管理，需要用户自己去管理，实现Sink的幂等。SparkStreaming中实现exactly-once的方法是：Source at-least-once+Sink Impedent。下面分别列举一下对各种不同Source，SparkStreaming所提供的支持。

#### 2.1 Checkpoint机制
Checkpoint提供了一种可以从失败中恢复的机制。其本身直接的目的并不是为了exactly-once，但对于HDFS、S3这类文件系统，Checkpoint机制可以提供at-least-once的支持。

Checkpoint可以保存运行时环境，包括Driver端的元数据：未完成的batch等。因此，在执行过程中程序异常退出，重新启动后可以通过checkpoint恢复之前的工作，重新执行这些未完成的job。

只不过有一个问题，SparkStreaming无法处理Job层面未完成的情况，也就是说一个job执行到一半，写了一部分数据到Sink，重新执行这个Job会导致数据的多次处理。基于这个原因，才需要Sink支持幂等。下同。
#### 2.2 WAL机制
SparkStreaming在1.2版本中引入了WAL机制，WAL机制是为了防止数据的丢失。在使用Receiver接收数据的时候，不同于HDFS，SparkStreaming此时是将接收到的数据以block的形式保存在内存中，并将block的信息通知到Driver，之后Driver再在这些Block上生成job。因此当程序异常退出后，这些保存在内存中的数据就会丢失，因此才会考虑先将数据刷写到可靠存储中，然后再做进一步处理，这就是WAL。

所以说，WAL提供了支持ack的Source的at-least-once的支持：收到数据并数据写到可靠文件系统中，然后再发送ack响应到Source，Source之后更新该Receiver对应的offset。这类Source包括Kafka、Kinesis等。这里说的Kafka指的是基于Receiver来接收数据的方式。

同样，WAL机制没有解决数据重复处理的问题，需要Sink的幂等支持

#### 2.3 Direct API
WAL机制通过降低了系统的吞吐量来获得了数据的at-least-once支持。但实际上，类似于Kafka这种非传统的消息队列，它同样提供了存储能力，类似于一个文件系统从一个文件的指定偏移处开始读取文件一样，我们同样可以基于Kafka中一个partition中各个record的offset获取到某个offset区间的数据。因此，只要把Kafka也当做一个类似HDFS的文件系统来处理的话，就不再需要WAL机制了。

在Direct API中，Driver首先获取到Source的最新的offset_1，结合当前保存的、已经处理到的offset_2，生成一个RDD。这里的RDD就是offset的区间内的数据，但真正读取数据是在Executor中具体执行到一个Action操作时才发生的，是不是跟文件系统一样了？

这里要注意的一个点就是对offset的保存，其实也叫做checkpoint，不过是Source offset的checkpoint，与Spark本身提供的checkpoint机制完全不是一回事。offset可以保存在任何可靠地位置，只要自己愿意去实现。那offset何时保存呢？推荐是在一个batch job完成后更新。这样其实也就用不到Spark本身提供的checkpoint机制了：offset更新了表示之前的数据被处理过了。但仍是建议开启Spark本身的Checkpoint，毕竟一部分已经完成的batch job，实际上没有必要重新执行一遍。

准确来说，Amazon Kinesis并没有Direct API。Direct API是在执行RDD计算中，由Executor主动去拉取数据，而不是有一个或多个Receiver负责从Source接收数据。Kinesis提供了KCL自动管理checkpoint，看似完成了相同的功能：自动向DynamoDB中更新offset，但想想还是有不少问题的，很容易造成数据丢失。Kinesis的KCL中有一个单独的线程根据给定的时间间隔不断的到DynamoDB中更新该Receiver对应的当前已经读取到的Kinesis的shard的offset。由于接收到了数据就更新offset，而不是处理后，就导致失败重启时仍有可能会丢失数据。因此对这种情况来说，仍然需要Spark本身的Checkpoint来支持：记录当前未完成的batch job。但由于Kinesis本身是基于Receiver模式的，因此在Receiver接收到数据形成Block，到Driver收到Block的元数据并将Block与batch job关联起来之间，假如DynamoDB中的offset更新了，Driver又挂了，此时就Spark本身的Checkpoint机制也无用，因为batch job根本就还没形成，都不知道自己跟谁挂钩的。不知道Kinesis是如何解决这个问题的，还是说需要用户自己解决。

#### 2.4 Sink的支持
上面说的其实都是SparkStreaming自身以及与上游Source的集成，使得对数据的处理能够满足at-least-once的要求。要想达到exactly-once的要求，还需要Sink的支持：要么是写入幂等的，要么实现写入的事务性。显然，key-value形式的存储系统都是支持幂等写入的，比如HBase、DynamoDB等。

比如我们使用的是DynamoDB作为SparkStreaming的Sink。DynamoDB的key使用的是：appID(小程序ID)作为hashKey，sessionID(会话ID)作为rangeKey。

对于一份相同的数据，Spark的RDD抽象保证了一定能够得到相同的结果。因此，加入一个job执行到一半失败了，就算此时写入了一部分数据到DynamoDB中，在程序从Checkpoint中恢复时，一样可以重新计算得到相同的结果，并更新到前面写入到DynamoDB中的数据。用这种方式，我们能够保证exactly-once。

总结来说，我们使用Kinesis+SparkStreaming提供的Checkpoint机制以及支持幂等写入的DynamoDB实现了exactly-once的要求。

### 3. 对Offset的管理
从上面的介绍中知道，要实现SparkStreaming对Source中数据的at-least-once，对Kafka这类Source来说，最重要的就是对Offset的管理。如何对Offset进行管理，是用户自己定义的，可以选用一下几种方式。

#### 3.1 使用SparkStreaming的Checkpoint机制
SparkStreaming本身提供了Checkpoint机制，存储了Driver端的元数据，以及当前完成的RDD的数据。Driver的元数据包括配置、要执行的操作、以及未完成的batch等。由于在DirectAPI中，RDD的定义是offset的range，从而使得失败中恢复执行replay数据的时候，只需要从Kafka之类的数据源重新读取即可。因此SparkStreaming本身的Checkpoint机制可以实现这一要求。

这一方法唯一一个问题就是：当用户程序升级后，导致序列化后的二进制文件发生变化，使得在recover的时候Spark不认识之前写入Checkpoint的数据，提示`Class Not Found`。

前面有提到过解决办法：删除Checkpoint文件夹下checkpoint开头的文件。(没实际测试过还)

### 3.2 其他存储系统中
可以将offset放到HBase或Zookeeper中等地方，或者直接使用Kafka进行管理。具体的代码参考可以参照[这篇文章](https://juejin.im/entry/5acd7224f265da237c693f7d)。


### 4. 参考文章
> [kafka-exactly-once](https://github.com/koeninger/kafka-exactly-once/blob/master/blogpost.md)
> [Kinesis-with-SparkStreaming](https://medium.com/@b23llc/exactly-once-data-processing-with-amazon-kinesis-and-spark-streaming-7e7f82303e4)
> [SparkStreaming与Kafka](https://zhuanlan.zhihu.com/p/30721699)
> [SparkStreaming与Kafka](https://blog.cloudera.com/blog/2015/03/exactly-once-spark-streaming-from-apache-kafka/)
