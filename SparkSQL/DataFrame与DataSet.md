### RDD、DataSet与DataFrame
这三者都是Spark平台下的弹性分布式数据集，为处理大型数据提供方便。RDD主要用在Spark中，DataSet/DataFrame是在Spark SQL中引入的，主要是为了处理结构化/半结构化的数据，包含了更多数据结构上的信息，以及将要在数据上执行的计算。在内部，Spark SQL会利用这些额外的信息，来做一些优化的工作。

Spark2.0之后，DataSet和DataFrame既能表示静态的、有限的数据，也能表示无限的、流式的数据。两种情况下入口都是SparkSession。

### DataFrame与DataSet区别
DataFrame/DataSet类似于一张表，每一行都是一条数据。在DataSet中，表中指定了各个字段的名称与类型，如：

```
scala> case class Person(name: String, age: Int)
scala> val personDS = Seq(Person("Max", 33), Person("Adam", 32), Person("Muller", 62)).toDS()
personDS: org.apache.spark.sql.Dataset[Person] = [name: string, age: int]

scala> personDS.show()
+------+---+
|  name|age|
+------+---+
|   Max| 33|
|  Adam| 32|
|Muller| 62|
+------+---+
scala> personDS.map(person => person.name + ": " + person.age).show
+----------+
|     value|
+----------+
|   Max: 33|
|  Adam: 32|
|Muller: 62|
+----------+
```

DataFrame具有和DataSet相同的API，区别是每一行的数据类型不同：每一行都是Row类型(实际上，`DataFrame`的定义格式就是`DataSet[Row]`)。所以，只有通过解析才能获取到各个字段的值。DataFrame在行的类型不确定(如多个case class)时会很有用。

```
scala> personDS.toDF
res14: org.apache.spark.sql.DataFrame = [name: string, age: int]
scala> res14.map{
     | case Row(name: String, age: Int) =>
     | println(name + age)
     | name
     | case _ =>
     | ""
     | }
res16: org.apache.spark.sql.Dataset[String] = [value: string]

scala> res16.show
Max33
Adam32
Muller62
+------+
| value|
+------+
|   Max|
|  Adam|
|Muller|
+------+
```

### RDD、DataSet和DataFrame的转化
1. DataSet转DataFrame、RDD
  只需要将case class封装成Row
  ```Scala
  import spark.implicits._
  val dataframe = dataset.toDF
  ```

  转RDD直接调用函数：
  ```Scala
  scala> val r1 = dataset.rdd
  r1: org.apache.spark.rdd.RDD[Person] = MapPartitionsRDD[166] at rdd at <console>:33

  scala> r1.collect
  res20: Array[Person] = Array(Person(Max,33), Person(Adam,32), Person(Muller,62))
  ```

2. DataFrame转DataSet、RDD
  需要提供case class的定义(或基本类型的定义)：
  ```Scala
  case class Person(name: String, age: Int)
  val dataset = dataframe.as[Person]
  // val dataset = dataframe.as[String]
  ```

  同样，直接调动函数转化为RDD
  ```Scala
  scala> val r2 = dataframe.rdd
  r2: org.apache.spark.rdd.RDD[org.apache.spark.sql.Row] = MapPartitionsRDD[170] at rdd at <console>:35

  scala> r2.collect
  res21: Array[org.apache.spark.sql.Row] = Array([Max,33], [Adam,32], [Muller,62])
  ```
3. RDD转DataFrame、DataSet

  ```Scala
  rdd.map {line => Person(...)}.toDS
  // rdd.map {line => Person(...)}.toDF
  ```
