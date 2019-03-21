在分布式集群中，很大的一个消耗就是数据的网络传输，特别是在数据中心，网络带宽资源会显得更加珍贵。在Spark中使用数据本地性，让计算跟着数据走，来最大程度的减少数据的网络传输。

### Spark任务执行流程
又要说一遍Spark的任务执行流程

![Spark任务提交流程](images/2019/02/spark任务提交流程.png)
在Driver中，每个Action操作都会触发一个job的提交。在一个job内部，涉及到若干个RDD的转化操作，根据RDD之间的依赖关系，形成了DAG图。在DAGScheduler中，会根据RDD依赖的类型(宽依赖还是窄依赖)将DAG分为若干个Stage。每个Stage中包含若干个task，称为taskset。taskset中task的个数根据RDD的partition数量决定：每个partition上都会启动一个task。然后TaskScheduler会将task分配到一个个的Executor中执行。具体分配到哪个Executor中，其中的一个判断依据就是人物执行的本地化：计算跟着数据走。

### 本地性的级别
在`TaskLocality`对象中定义了本地性的几个级别

```Scala
object TaskLocality extends Enumeration {
  val PROCESS_LOCAL, NODE_LOCAL, NO_PREF, RACK_LOCAL, ANY
}
```
- `PROCESS_LOCAL`，进程本地化，待处理的数据和task会启动在同一个Executor上的jvm中，从而完全不需要数据的移动
- `NODE_LOCAL`，节点本地化，待处理的数据和task启动在同一个Worker节点的不同Executor的jvm中，涉及到进程间的数据传输
- `NO_PREF`，如果数据的位置没有影响，或者Spark无法决定如何对任务进行本地化
- `RACK_LOCAL`，机架本地化，待处理的数据和task启动在同一个机架上不同机器的Executor中，涉及机架内部机器间的网络传输
- `ANY`，不同机架间的机器上，完全没有本地化，网络传输文件

可以在WebUI上查看本地化的级别，如：

![WebUI上查看本地化级别](/images/2019/02/webui上查看本地化级别.png)
### task的分配策略
Spark任务本地化的目的是减少数据传输的消耗，因此根据上面的级别很容易得出优先级。TaskScheduler在分配task到某个Executor后，在该Executor接受并执行该task前，会等待一段时间，如果这个Executor由于正在执行任务或其他的原因，在TaskScheduler等待的这段时间内没有能够开始执行这个task，那么TaskScheduler就会把本地化的级别降低，然后重新尝试分配给另一个(有没有可能是同一个？)Executor。

### 参数配置
TaskScheduler等待的时间的长短有几个参数来配置
- `spark.locality.wait`
  - **意义**，为执行数据本地化任务，在将本地化级别降低前所等待的时间段。各个本地化级别之间等待的时间长度都用这个值，但也可以为每个本地化等待的时间配置自己各自独特的值(下面几个参数)。如果任务执行时间过长，并且本地化性能差，那么应该调大这个值。不过这个默认值一般都可以工作的很好
  - **默认值**，3s
- `spark.locality.wait.process`
  - **意义**，为执行进程本地化，所可以等待的时间
  - **默认值**，使用`spark.locality.wait`设置的值
- `spark.locality.wait.node`
  - **意义**，为执行节点本地化，所可以等待的时间
  - **默认值**，使用`spark.locality.wait`设置的值
- `spark.locality.wait.rack`
  - **意义**，为执行机架本地化，所可以等待的时间
  - **默认值**，使用`spark.locality.wait`设置的值
