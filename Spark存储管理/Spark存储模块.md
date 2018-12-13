
[BlockManager](http://shiyanjun.cn/archives/1641.html)

[原文链接](https://www.jianshu.com/p/58288b862030)

---

## 存储模块
Spark的storage模块管理着Application在运行过程中产生的各种数据，将那些在内存或磁盘、在本地或远程存取数据的功能封装了起来，实现了数据与具体物理存储(堆内外、磁盘、内存)之间的解耦合。

### Block
Storage模块在逻辑上以Block为基本存储单位，并负责对这些Block进行管理。Storage模块中的Block实际上就对应了RDD中的Partition：RDD的每个Partition经过处理后唯一对应一个Block(BlockId的格式为rdd_RDD-ID_PARTITION-ID)。每个BLock由唯一的BlockID(`org.apache.spark.storage.RDDBlockId`)标识，格式为`rdd_ + rddId + "_" + splitIndex`，如`rdd_0_0`。

![RDD的Partition与Block的逻辑关系](/images/2018/12/rdd的partition与block的逻辑关系.png)

### 存储模块架构
Storage模块是Master/Slave架构：即Driver端的BlockManager为Master，Executor端的BlockManager为Slave。

![Storage模块](https://upload-images.jianshu.io/upload_images/35301-d83f9c38f26f5752.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

Master负责整个Application中Block元数据信息的维护和管理，Slave需要将Block的更新信息上报Master，同时接收Master的命令，如新增、删除一个RDD。

### BlockManager

`org.apache.spark.storage.BlockManager`是storage模块与其他模块交互最主要的类，它提供了读写Block的接口。Driver和Executor上Block数据的管理都是由这个类完成的。

上层的计算都是依据RDD的Partition进行的，storage模块里的所有操作都是和Block相关的。Partition和Block是如何进行转换的？

##### 1. RDD Partition -> BLock
这一过程发生在RDD的缓存。RDD Partition中的数据使用迭代器Iterator进行访问，迭代器可以获取到Partition中的每一条记录Record。这些Record对象实例占据了Spark内存模型中的Other部分。并且同一个Partition的不同Record之间在存储空间中是地址不连续的。

RDD的缓存由Storage模块完成，实现了RDD与物理存储(堆内外、磁盘、内存)之间的解耦合。在缓存过程中，Storage模块会把原本的Partition变为Block，把Block存放在堆内或堆外内存的Storage部分中的一块连续的空间中。在Spark中，由原本存放不连续的Partition变为存放连续的Block的过程称为Unroll。

在Unroll过程中，因为无法保证Storage部分有足够的内存，因此在Unroll前需要先申请内存，并将这块内存占据。对于非序列化的Partition而言，使用Iterator读取的是一条一条的Record，在每处理一条Record前都需要估算需要用到的空间，并进行申请和占据。空间不足，可以释放已有的缓存。对于序列化后的Partition，其所用到的空间可以一次性申请。

![Spark Unroll](https://upload-images.jianshu.io/upload_images/35301-63923cd1946dd929.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)


###### 2. Block -> RDD Partition
RDD Partition中的数据使用迭代器Iterator进行访问，迭代器可以获取到Partition中的每一条记录Record。RDD类的iterator函数定义为：
```
final def iterator(split: Partition, context: TaskContext): Iterator[T] = {
    if (storageLevel != StorageLevel.NONE) {
      getOrCompute(split, context)
    } else {
      computeOrReadCheckpoint(split, context)
    }
  }
```
如果当前RDD的StorageLevel不为NONE，首先会判断是否有缓存，有缓存则直接读取，没有缓存需要计算Block并对这个Block进行缓存。代码在`getOrCompute`函数中：
```Scala
SparkEnv.get.blockManager.getOrElseUpdate(blockId, storageLevel, elementClassTag, () => {
  readCachedBlock = false
  computeOrReadCheckpoint(partition, context)
})
```
使用BlockManager的getOrElseUpdate函数来获取。如果没有找到缓存，会使用提供的偏函数，计算RDD`computeOrReadCheckpoint`。
