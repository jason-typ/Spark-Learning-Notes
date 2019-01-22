对[原文](https://github.com/lw-lin/CoolplaySpark/blob/master/Structured%20Streaming%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%E7%B3%BB%E5%88%97/1.1%20Structured%20Streaming%20%E5%AE%9E%E7%8E%B0%E6%80%9D%E8%B7%AF%E4%B8%8E%E5%AE%9E%E7%8E%B0%E6%A6%82%E8%BF%B0.md)的学习

任何计算引擎都需要三个部分：数据输入、数据处理和数据输出。在Spark中如下图：

![](/images/2019/01/Screen Shot 2019-01-17 at 8.54.16 PM.png)

要大致了解Structured Streaming，起码需要能够回答以下几个问题：

1. 支持哪些Source？为什么？对于无限增长的表，如何实现的增量查询？
2. 支持哪些Sink？为什么？
3. 如何支持的exactly once？
3. 如何触发的计算？
4. 如何输出到Sink？
5. 事件时间与处理时间
6. 窗口、waterMark

要回答这些问题，先得看一下Spark的持续查询引擎。


## StreamExecution：持续查询的运转引擎


### 1. StreamExecution的初始状态

StreamExecution中几个重要的成员变量：
- uniqueSources，数据源，如Kafka等
- logicalPlan，SQLparser对计算逻辑解析后的结果
- sink，写出数据的接收端
- currentBatchId，当前执行的batch job的id
- commitLog，记录已经完成的batch ID的log。用于检查某个batch是否被执行完毕，并且输出已经被写入到sink中，从而不需要再次执行。这个log用于重启时检查，定位从哪个batch开始重新执行
- offSetLog，一个write-ahead-log，记录当前batch的offset


### 2. StreamExecution持续查询

![持续查询](https://github.com/lw-lin/CoolplaySpark/blob/master/Structured%20Streaming%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%E7%B3%BB%E5%88%97/1.imgs/110.png)

1. StreamExecution 通过 Source.getOffset() 获取最新的 offsets，即最新的数据进度；
2. StreamExecution 将 offsets 等写入到 offsetLog 里
3. StreamExecution 构造本次执行的 LogicalPlan
4.

我们知道Structured Streaming中将输入抽象做一个可以无限增长的表格，所以对于这个表的查询一定是需要是增量形式的。
![StreamExecution](https://github.com/lw-lin/CoolplaySpark/blob/master/Structured%20Streaming%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%E7%B3%BB%E5%88%97/1.imgs/100.png)
