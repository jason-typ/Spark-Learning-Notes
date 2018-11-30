# SparkDriver

#### Driver的初始化
Driver的初始化，其实也就是SparkContext的初始化，一个Driver可以看做就是一个SparkContext。SparkContext的初始化需要一个`SparkConf`的实例：

```
val conf = new SparkConf().setAppName("WordCount")
// val conf = new SparkConf().set("spark.app.name", "WordCount")
val sc = new SparkContext(conf)
```

SparkConf内部维护一个HashMap，保存各种属性信息。我们只需要调用上层提供的接口来设置各种属性配置就可以了。在spark-shell中，默认已经给我们生成了一个SparkContext的实例，叫做sc。

#### Driver中的调度

从之前的介绍来看，SparkContext的调度分为两个部分：DAGScheduler和TaskScheduler。DAGScheduler负责将作业(DAG图)分解为若干stage，TaskScheduler负责将每个Stage中的任务提交到集群中。

###### stage的划分

根据RDD之间的依赖关系得到DAG图后，DAGScheduler会将DAG图划分为不同的stage，划分的依据就是RDD之间依赖的类型。

这里比较困惑，看网上好多解释。暂时的理解如下，以后发现错误再改。

- NarrowDependency 子RDD的每个分区都只依赖于父RDD的一个分区
- ShuffleDependency 子RDD的某个分区依赖于父RDD的多个分区

对于NarrowDependency，由于子RDD的分区只会依赖于父RDD的一个分区，不涉及到不同分区之间的混合，因此不需要等待所有父RDD的分区都已准备完毕就可以执行，因此对于多个窄依赖RDD之间，可以形成流水线式的操作。

stage划分后会有两种类型：

- ResultStage 最终的Stage，没有输出结果。
- ShuffleMapStage 非最终的Stage，作为后续Stage的输入。Stage的划分是根据ShuffleDependency来断开的，因此一个Stage的输出一定是会经过Shuffle的，结果就是ShuffleMapStage

一次一个Job中至少会有一个ResultStage。
