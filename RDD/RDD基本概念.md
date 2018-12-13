---
title: RDD基本概念
categories: Spark基础
tags: Spark
---

### RDD概念
Spark中的RDD是一个不可变的分布式对象的集合。RDD可以被分为多个分区，每个分区就是一个数据集片段(只读的分区记录集合)。并且这些分区可以分布在集群的不同节点上，从而保证并行计算。

### RDD的创建
Spark的RDD分为两种：
* parallelizedcollections
	基于驱动程序中的对象集合（如list和set），使用`parallelize`方法创建：
	```
	val a = sc.parallelize(List(1, 2, 3, 4, 5))
	```
	变量a就是一个RDD，打印它的类型可以看到为`org.apache.spark.rdd.ParallelCollectionRDD`。这种方法把数据集先放在了一台机器的内存上，因此实际上用到的不多，除了自己测试。
* Hadoopdatasets
	基于HDFS或者其他其他Hadoop支持的文件存储系统（外部存储）创建distributed dataset,如创建一个基于文本文件的DistributedDataset：
	```
	val lines = sc.textFile("file:///Users/tang/Documents/docker")
	```
	`lines`的类型就是`org.apache.spark.rdd.MapPartitionsRDD`

### RDD分区(Partition)
Spark会在每个partition上启动一个task。

1. 分区的个数
分区可以增加程序的并行度：同一个RDD不同分区的数据可以同时进行计算。注意，RDD的分区，与HDFS中的block不是一个概念。要对RDD进行分区
	- 在创建RDD时，指定分区数目，如`val textFile = sc.textFile(inputFile, 2)`
	- 转换得到中间RDD时，调用`repartition(num)`方法

	如果没有设置分区数目，Spark有个配置选项`spark.default.parallelism`设置了RDD默认的分区数(这个设置的默认值根据不同的部署模式会有所不同)。某个RDD的分区，可以调用方法`textFile.partitions`得到，方法返回的类型是数组`Array[org.apache.spark.Partition]`，分区的数目就可以通过`textFile.partitions.size`得到。(HDFS上，每个block都会默认建立一个分区)

	对RDD进行重新分区需要进行数据的混洗，因此其开销是很大的。一般在分组或聚合时进行重新分区。
2. 分区的依据
Spark提供了分区的抽象类`Partitioner`，并提供了抽象类的单例对象(默认的分区依据)。要实现定制化的分区函数，需要定义一个类继承自`Partitioner`，并在RDD上调用`partitionBy()`函数时传入这个类的实例

3. 测试
	```
	import org.apache.spark.{Partitioner}
	scala> class MyPartitioner(num: Int) extends Partitioner {
     | override def numPartitions: Int = num
     | override def getPartition(key: Any): Int = {
     | key.toString.toInt % 10
     | }
     | }

	scala> val data = sc.parallelize(1 to 10, 5)
	scala> data.partitions.size
	res18: Int = 5
	scala> data.map((_, 1)).partitionBy(new MyPartitioner(10)).map(_._1).saveAsTextFile("hdfs://jason:9000/user/tang/output")
	// 结果被写入到10个不同的文件中
	```

### RDD的操作
RDD支持两种类型的操作：转化(transformation)操作和行动(action)操作。

转化操作和行动操作的区别在于计算RDD的方式不同。转换操作是返回一个新的RDD的操作，并且转化操作不会触发实际的计算（惰性求值）；行动操作会触发实际的计算，它会强制计算求值过程中需要用到的RDD转化操作，并将结果返回给驱动程序或写入外部系统。

查看一个特定函数是转化操作还是行动操作，只需要看它的返回类型：如果返回RDD，则就是一个转化操作；如果返回其他的，就是行动操作。

1. 转化操作
	一个简单的filter操作如下：
	```
	inputRdd = sc.textFile("README.md")
	val errorsRdd = inputRdd.filter((line) => line.contains("error"))
	val warningsRdd = inputRdd.filter((line) => line.contains("warnings"))
	badlinesRdd = errorsRdd.union(WarningsRdd)
	```
2. 行动操作
	一个简单的filter操作如下：
	```
	println(badlinesRdd.count())
	badlinesRdd.take(10).foreach(println)
	```
	RDD还有一个`collect`函数，可以用来获取整个RDD中的数据（全部数据都放在一台机器的内存上，要放得下才行，因此大多数情况下无法使用）。

### RDD的持久化缓存
每当我们调用一个新的行动操作时，整个RDD都可能会从头开始计算。为避免这种低效的行为，我们可以把需要用到的中间结果(RDD)进行持久化保存。

持久化保存使用`persist(level)`(或`cache`)函数。根据需要不同，可以设置RDD持久化的级别(比如RDD缓存在内存中还是硬盘上)。默认情况下，`persist()`会把数据以反序列化的形式(`MEMORY_ONLY`)缓存在JVM的堆空间中。持久化级别如下：
- MEMORY_ONLY，RDD作为`反序列化`的对象缓存到JVM堆内存中。如果堆中放不下了，按照LRU原则，替换缓存中的内容
- MEMORY_ONLY_SER，RDD序列化后缓存到JVM堆内存中
- MEMORY_AND_DISK，堆内存中放不下了，存放到硬盘上
- MEMORY_AND_DISK_SER
- DISK_ONLY

反序列化对象存储会耗费较高的存储空间，序列化存储需要CPU的计算时间来完成转换。

`persist`本身不会触发计算操作，它只是一个标记，标记这个RDD在计算出结果后进行缓存。还有一个`cache`方法，等价于`persist(MEMORY_ONLY)`。

另外，还有一个`unpersist()`方法，用于手动将持久化的RDD从缓存中移除。

### 函数传递

### 不同RDD类型之间的转化
