
### RDD概念

Spark中的RDD是一个不可变的分布式对象的集合。RDD可以被分为多个分区，每个分区就是一个数据集片段。并且这些分区可以分布在集群的不同节点上，从而保证并行计算。

### RDD的创建
Spark的RDD分为两种：

* parallelizedcollections

	基于驱动程序中的对象集合（如list和set），使用`parallelize`方法创建：

	<pre>
	val a = sc.parallelize(List(1, 2, 3, 4, 5))
	</pre>
	变量a就是一个RDD，打印它的类型可以看到为`org.apache.spark.rdd.ParallelCollectionRDD`。这种方法把数据集先放在了一台机器的内存上，因此实际上用到的不多，除了自己测试。

* Hadoopdatasets

	基于HDFS或者其他其他Hadoop支持的文件存储系统（外部存储）创建distributed dataset,如创建一个基于文本文件的DistributedDataset：

	<pre>
	val lines = sc.textFile("/Users/tang/Documents/docker")
	</pre>
	`lines`的类型就是`org.apache.spark.rdd.MapPartitionsRDD`

### RDD的操作

RDD支持两种类型的操作：转化(transformation)操作和行动(action)操作。转化操作会由一个RDD生成另一个RDD；行动操作会计算出一个结果。

转化操作和行动操作的区别在于计算RDD的方式不同。转换操作是返回一个新的RDD的操作，不会触发实际的计算（只有在行动操作中需要用到这些RDD时才会真正执行计算）；行动操作则是向驱动程序返回结果或把结果写入外部系统的操作，会触发实际的计算。

查看一个特定函数是转化操作还是行动操作，只需要看它的返回类型：如果返回RDD，则就是一个转化操作；如果返回其他的，就是行动操作。

1. 转化操作

	一个简单的filter操作如下：

	<pre>
	inputRdd = sc.textFile("README.md")
	val errorsRdd = inputRdd.filter((line) => line.contains("error"))
	val warningsRdd = inputRdd.filter((line) => line.contains("warnings"))
	badlinesRdd = errorsRdd.union(WarningsRdd)
	</pre>
2. 行动操作

	一个简单的filter操作如下：

	<pre>
	println(badlinesRdd.count())
	badlinesRdd.take(10).foreach(println)
	</pre>

	RDD还有一个`collect`函数，可以用来获取整个RDD中的数据（全部数据都放在一台机器的内存上，要放得下才行，因此大多数情况下无法使用）。

### RDD常用操作

1. 常用转化操作

* `filter()`接收一个函数，将RDD中满足该函数的元素放入新的RDD中并返回。

* `map()`接收一个函数，将函数作用于RDD中的每个元素中，并将结果放入新的RDD中，返回新的RDD。
* `flatmap()`
* `distinct()` 去重
* `union(other)`操作，返回一个包含两个RDD中所有元素的RDD。`union`操作不会去除重复元素：如果两个RDD中有重复元素，结果RDD中会包含所有的元素。
* `intersection(other)`方法，返回两个RDD中都有的元素。`intersection`在运行时会去掉所有重复的元素，包括自身里面的重复元素。因此，`intersection`操作性能差，因为要通过网络混洗数据来发现相同数据。
* `subtract(other)`方法，接收另一个RDD作为参数，返回一个由只存在于第一个RDD而不存在与第二个RDD中的所有元素组成的RDD。和`intersection`一样，也需要数据混洗
* 笛卡尔积，`cartesian(other)`方法

2. 常用行动操作

	* `reduce()`，接收一个函数作为参数。reduce将这个函数作用在序列上，并将结果继续和序列的下一个元素做累积计算。如求和操作`val sum = rdd.reduce((x, y) => x + y)`
	* `fold()`
	* `aggregate()`

3. 从结果RDD中获取数据

	数据返回驱动程序
	* collect
	* take
	* top


### RDD的持久化缓存


### 关于RDD

RDD是只读的分区记录集合，RDD可以分为多个分区，每个分区就是一个数据集片段（分区可以保存在不同的节点上，用于并行计算）。Spark的RDD分为两种：

* parallelizedcollections

	基于现有的Scala collection集合构建, 使用`parallelize`方法创建：

	<pre>
	val conf = new SparkConf()
	val sc = new SparkContext(conf)

	val a = sc.parallelize(List(1, 2, 3, 4, 5))
	</pre>
	变量a就是创建出来的一个RDD，打印它的类型可以看到为`org.apache.spark.rdd.ParallelCollectionRDD`

* Hadoopdatasets

	基于HDFS或者其他其他Hadoop支持的文件存储系统创建distributed dataset，如创建一个基于文本文件的DistributedDataset：

	<pre>
	val textFile = sc.textFile("/Users/tang/Documents/docker")
	println(textFile.count())
	println(textFile.getClass)
	</pre>

	`textFile`的类型就是`org.apache.spark.rdd.MapPartitionsRDD`
