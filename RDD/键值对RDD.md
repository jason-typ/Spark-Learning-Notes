# 键值对RDD

### 1. PairRDD的创建
同一般RDD一样，可以从外部文件系统中读取，也可以通过内存中的数据集。从外部文件系统读取时，有些文件格式已经是Pair的格式，不需要转化，不是Pair格式的，可以通过map函数进行转化。

### 2. PairRdd的转化操作

- keys()，返回仅包含RDD中键的RDD
- values()，返回仅包含RDD中值的RDD


###### 2.1. 聚合操作
聚合PairRDD中具有相同键的值是很常用的操作。一些常用的聚合函数有：

- mapValues(func)，对PairRDD中的每个值应用函数func，键保持不变
- flatMapValue(func)，将func作用在PairRDD中的每个值上(flatMap要求func的返回值是一个可迭代对象)，然后对于返回的可迭代对象中的每个值，都与原先的键值组成一个新的Pair。如
  ```Scala
  scala> val pair = sc.parallelize(List(1)).map((_, 5))
  pair: org.apache.spark.rdd.RDD[(Int, Int)] = MapPartitionsRDD[1] at map at <console>:24
  scala> pair.collect.foreach(println)
  (1,5)
  scala> val res = pair.flatMapValues(x => x to 10)
  res: org.apache.spark.rdd.RDD[(Int, Int)] = MapPartitionsRDD[2] at flatMapValues at <console>:25
  scala> res.collect.foreach(println)
  (1,5)
  (1,6)
  (1,7)
  (1,8)
  (1,9)
  (1,10)
  ```
- reduceByKey(func)，对RDD中具有相同键值的元素执行reduce操作，结果与键作为新RDD中的元素
- foldByKey(func)，与reduceByKey一样的操作，只不过提供了个初始值
- combineByKey()，基于键进行聚合，多个参数分别对应了聚合操作的不同阶段。这个函数的定义为：
  ```Scala
  def combineByKey[C](
      createCombiner: V => C,
      mergeValue: (C, V) => C,
      mergeCombiners: (C, C) => C): RDD[(K, C)]
  ```
  处理数据的过程。这个函数会遍历**分区**中的所有数据。这个数据的键要么之前没有遇到过，要么之前已经遇到过。如果这是一个新的元素，combineByKey会调用createCombiner函数，来创建一个那个键对应的累加器的初始值。如果这个键在当前分区的处理中已经遇到过，combineByKey会调用mergeValue方法，将该键对应的累加器的当前值与这个新的值进行合并。以上的处理都是以分区为单位的，每个分区的处理都是独立的，因此一个RDD中，同一个键可以有多个累加器。最后就是调用mergeCombiners方法将多个分区的累加器的当前值进行合并.

例子：
1. 统计PairRDD中每个键对应的平均值
  ```Scala
  rdd.mapValues((_, 1)).reduceByKey((x, y) => (x._1 + y._1, x._2 + y._2))
  ```
  首先把value变成元组，表示出现的次数。然后按照key进行归并，对元组进行操作。元组的第一个值表示原本键对应的值，第二个值表示出现的次数。这样最后结果中的元组的第一个值除以第二个值就是平均数了
2. 单词计数
  ```Scala
  val input = sc.textFile("s3://...")
  val words = input.flatMap(x => x.split(" "))
  val result = words.map(x => (x, 1)).reduceByKey(_ + _)
  // words.countByValue()
  ```
  `words.countByValue`同样完成计数功能，但是在数据量大的情况下，还是需要使用`map`+`reduceByKey`的方法
3. 使用reduceByKey统计PairRDD中每个键对应的平均值
  ```Scala
  rdd.reduceByKey(
    v => (v, 1),        // 创建累加器
    (acc: (Int, Int), v) => (acc._1 + v, acc._2 + 1), // 用值更新累加器
    (acc1: (Int, Int), acc2: (Int, Int)) => (acc1._1 + acc2._1, acc1._2 + acc2._2)      // 累加器的合并
    )
  ```
###### 2.2. 数据分组
将数据按照键进行分组也很常用
- groupByKey()，对RDD中具有相同键值的元素进行分组(同一个键的所有值放到一个列表(Iterable)里面)。上面在`combineByKey`中介绍了按键进行聚合的执行过程：现在分区内合并，然后各个分区再合并。`groupByKey`与`reduceByKey`的一个区别就是：`reduceByKey`会现在本分区内进行合并，合并结果再进行分区间合并，所以分区间数据混洗涉及的数据交换量会少一些；而`groupByKey`没有分区内的合并，因此在分区间合并时涉及的数据交换没有减少，因此效率会更低。

###### 2.3. 数据连接
将两个PairRDD连接也是很常用的操作。连接分为多种类型：内连接、外连接等。

- join(RDD)，内连接，两个RDD中都有的键才会输出
  ```Scala
  scala> val pair1 = sc.parallelize(Array(("a", 1), ("a", 2), ("b", 1)))
  pair1: org.apache.spark.rdd.RDD[(String, Int)] = ParallelCollectionRDD[0] at parallelize at <console>:24

  scala> val pair2 = sc.parallelize(Array(("a", "test")))
  pair2: org.apache.spark.rdd.RDD[(String, String)] = ParallelCollectionRDD[1] at parallelize at <console>:24

  scala> pair1.join(pair2).collect.foreach(println)
  (a,(1,test))
  (a,(2,test))
  ```
- leftOuterJoin(RDD)，左外连接，包括源RDD中的所有元素，键对应的值是两个RDD中键对应的值的组合。如果在目标RDD中没有该键，则结果为None，如
  ```Scala
  scala> pair1.join(pair2).collect.foreach(println)
  (a,(1,test))
  (a,(2,test))

  scala> pair1.leftOuterJoin(pair2).collect.foreach(println)
  (b,(1,None))
  (a,(1,Some(test)))
  (a,(2,Some(test)))
  ```
- rightOuterJoin(RDD)，右外连接，与左外连接作用相同，方向相反，`A.leftOuterJoin(B)`等价于`B.rightOuterJoin(A)`。

###### 2.4. 数据排序
排序也很常用。

- sortByKeys()，返回按照键排序过的RDD，默认升序(传入false降序)
- sortBy(value)，`sortByKeys`只能按照键来排序，使用这个函数可以按照任意给定的字段进行排序

### 3. PairRDD的行动操作
所有基础RDD支持的行动操作都在PairRDD上可用，除此之外，PairRDD还提供了一些其他的行动操作：

- countByKey，对每个键对应的元素分别计数(如单词计数)
  ```Scala
  scala> val text = sc.textFile("hdfs://jason:9000/user/tang/README.md")
  text: org.apache.spark.rdd.RDD[String] = hdfs://jason:9000/user/tang/README.md MapPartitionsRDD[9] at textFile at <console>:24

  scala> text.flatMap(line => line.split(" ")).map(word => (word, 1)).countByKey
  ```
- collectAsMap，将结果以映射的形式返回，方便查询
  ```Scala
  scala> pair1.collectAsMap
  res4: scala.collection.Map[String,Int] = Map(b -> 1, a -> 2)
  ```
- lookup(key)，返回给定键对应的所有值
