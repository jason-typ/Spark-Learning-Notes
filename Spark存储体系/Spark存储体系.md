### 存储系统

## Spark内存存储

Spark优先使用各节点的内存作为存储，当内存不足时才会使用磁盘。这极大减小了磁盘的IO，提高了执行效率，因此Spark才会适用于实时计算、迭代计算。

### 存储内存管理

### RDD持久化缓存
### 1. RDD持久化机制

弹性分布式数据集（RDD）作为Spark最根本的数据抽象，是只读的分区记录（Partition）的集合。task在启动之初读取一个分区时，首先会判断这个分区是否已被持久化。如果没有持久化，则会根据RDD之间的血缘关系，重新计算。为避免重复的计算，需要将RDD缓存。缓存有多种级别，默认使用的是`MEMORY_ONLY`，即保存在内存中。

RDD的持久化由Storage模块负责。Storage模块实现了RDD与物理存储之间的解耦。在对RDD进行持久化时，Spark定义了几种持久化级别，由`org.apache.spark.storage.StorageLevel`来定义。其中的存储级别是几个变量的组合：

```Scala
class StorageLevel private(
    private var _useDisk: Boolean,    磁盘
    private var _useMemory: Boolean,  堆内内存
    private var _useOffHeap: Boolean, 堆外内存
    private var _deserialized: Boolean, 是否为非序列化
    private var _replication: Int = 1)  副本个数
```

可以看出存储级别从三个维度定义了RDD的Partition（同时也就是Block）的存储方式：
- 存储位置：磁盘／堆内内存／堆外内存。如MEMORY_AND_DISK是同时在磁盘和堆内内存上存储，实现了冗余备份。OFF_HEAP则是只在堆外内存存储，目前选择堆外内存时不能同时存储到其他位置。
- 存储形式：Block缓存到存储内存后，是否为非序列化的形式。如MEMORY_ONLY是非序列化方式存储，OFF_HEAP是序列化方式存储。
- 副本数量：大于1时需要远程冗余备份到其他节点。如DISK_ONLY_2需要远程备份1个副本

### 2. RDD缓存过程
RDD在缓存到Storage内存(或磁盘)之前，Partition中的数据使用迭代器(Iterator)来访问。通过Iterator可以获取到Partition中的每一条数据项(Record)。这些Record对象占用的是JVM中的Other部分的空间，并且同一Partition中的不同Record间的空间并不连续。

RDD在缓存到Storage内存时，会将Partition转换成Block。Block在堆内内存或堆外内存的Storage中会占据一块连续的空间。将Partition由不连续的存储空间转换为连续存储空间的过程，Spark称之为展开(Unroll)。Block有序列化和非序列化两种格式。

每个Executor的Storage模块用一个链式Map结构(LinkedHashMap)来管理堆内和堆外存储内存中所有Block对象的实例。对这个LinkedHashMap的新增和删除，间接的记录了内存的申请和释放。

在Unroll之前需要先确认Storage中存在足够大的存储空间，因此当前的task会通过MemoryManager先进行空间申请，并占据该空间，空间不足，则Unroll会直接失败。对于非序列化的Partition，会在遍历Record的时候依次申请，即每读取一条Record会对所需要的空间进行估算并进行空间申请。对于序列化的Partition，可以对其所需空间进行累加计算，一次性申请。


### 执行内存管理
#### 1. 多任务之间的分配
Executor内运行的任务共享执行内存
### 调度系统

调度系统主要由DAGScheduler和TaskScheduler组成，它们都内置在SparkContext中。
