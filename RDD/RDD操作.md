# RDD操作

### RDD常用操作API
1. 常用转化操作
	- `map(func)`，map函数接收一个函数，把这个函数作用于RDD中的每个元素，将函数的返回结果作为结果RDD中对应的元素
	- `filter(func)`，接收一个函数，将RDD中满足该函数的元素放入新RDD中返回
	- `flatMap(func)`，与map类似，多了一个展平操作。并且对func的类型有所要求。与Scala中常用的高阶函数都是一样的
2. 伪集合操作
	虽然RDD不是严格意义上的集合，但它也支持很多集合操作：
	- `distinct()`，去重，保证每个元素只有一份。去重动作开销很大，因为RDD的不同分区可以分布在不同的网络节点上，去重需要对所有数据通过网络进行混洗(shuflle)
	- `union(RDD)`，并集，将两个RDD中的所有元素合并到一个RDD中。注意，RDD的union操作不会去除(重复)元素
	- `intersection(RDD)`，交集，将两个RDD中共有的元素取出放到结果RDD中。该操作会去除重复的元素，因此会有数据的混洗，所以同`distinct`操作一样，开销很大。
	- `subtract(RDD)`，从第一个RDD中去除两个RDD都包含的元素(只存在于第一个RDD而不存在与第二个RDD中的所有元素组成的RDD)。也需要数据混洗
	- `cartesian(RDD)`，笛卡尔积，返回所有可能的(a, b)对组成的RDD。其中a来自第一个RDD，b来自第二个RDD。大规模的笛卡尔积开销巨大
3. 行动操作
	- `count()`，返回RDD中的元素个数
	- `first()`，返回RDD中的第一个元素
	- `reduce(func)`，与一般reduce一样用法
	- `fold(value)(func)`，与reduce一样，只不过提供了一个第一次迭代中用到的初始值
	- `aggregate()`
4. 从结果RDD中获取数据
	- `collect()`，将结果RDD中的所有内容返回给Driver，需要保证RDD的内容不会在内存中放不下
	- `take(n)`，返回RDD中的n个元素，无法在上层控制返回RDD哪个分区的数据
	- `top(n)`，如果为数据定义了顺序，就可以使用这个函数从RDD中获取前n个元素
	- `foreach(func)`，对RDD中的每个元素执行func，如println。但要注意，由于代码真正是在Executor的线程中执行的，因此对于println这类打印语句并不会出现在提交程序的Driver的终端中，而是会输出在Executor的标准输出/错误里。要想在Driver端的终端输出，还是需要用到`collect`或`take`函数

### shuflle
![Spark Shuffle](https://umbertogriffo.gitbooks.io/apache-spark-best-practices-and-tuning/content/stages1.png)

大规模数据集下RDD会被分为多个区，每个分区上执行一个task，从而方便并发的计算。当执行一个类似`reduceByKey`的操作时，我们需要把同一个key对应的所有的value都聚合起来。由于具有相同key的数据可能分布在不同的分区中，因此这一过程就需要计算所有分区中的数据。混合多个分区的数据的过程就叫做shuffle。由于shuffle会涉及到大量的磁盘读写、序列化/反序列化、网络数据交换，因此其代价是比较昂贵的。

引起shuffle的操作包括`repartition`，`ByKey`操作如`groupByKey`、`reduceByKey`(但不包括计数)，以及`join`操作。
