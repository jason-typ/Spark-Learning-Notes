[Spark内存管理详解(上)](https://www.jianshu.com/p/3981b14df76b)

[Spark内存管理详解(下)](https://www.jianshu.com/p/58288b862030)

[Spark Block存储管理](http://shiyanjun.cn/archives/1641.html)

[Apache Spark 统一内存管理模型详解](https://www.iteblog.com/archives/2342.html#Off-heap_Memory)


Spark存储结构分为两层：内存存储和磁盘存储。磁盘存储用于在内存不够时进行辅助存储。

Spark的内存分为两部分：堆内内存和堆外内存。

### 堆内内存

![堆内内存](https://www.ibm.com/developerworks/cn/analytics/library/ba-cn-apache-spark-memory-management/image004.png)

堆内内存可以分为四块：
- **Execution内存**：主要用于存放 Shuffle、Join、Sort、Aggregation 等计算过程中的临时数据
- **Storage内存**：主要用于存储 spark 的 cache 数据，例如RDD的缓存、unroll数据
-

#### UnifiedMemoryManager
Spark1.6之后引入统一内存管理机制。与静态内存管理的主要区别是：Storage内存和Execution内存的分配不再是固定的，可以动态的占用对方的空闲区域。

![](https://upload-images.jianshu.io/upload_images/3009881-a72cb0282275305b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/932/format/webp)


堆内内存的大小由`spark.executor.memory`参数配置，Executor内运行的并发任务共享这些内存。

堆内内存根据用途分为两种类型：Storage内存和Execution内存。

- Storage内存
  在任务中，用于缓存RDD和广播变量
- Execution内存
