
[BlockManager](http://shiyanjun.cn/archives/1641.html)

[原文链接](https://www.jianshu.com/p/58288b862030)

---

## 存储模块
Spark的storage模块管理着Application在运行过程中产生的各种数据，将那些在内存或磁盘、在本地或远程存取数据的功能封装了起来，实现了数据与具体物理存储(堆内外、磁盘、内存)之间的解耦合。

### Block
Storage模块在逻辑上以Block为基本存储单位，并负责对这些Block进行管理。Storage模块中的Block实际上就对应了RDD中的Partition：RDD的每个Partition经过处理后唯一对应一个Block。每个BLock由唯一的BlockID(`org.apache.spark.storage.RDDBlockId`)标识，格式为`"rdd_" + "rddId" + "_" + "partitionId"`，如`rdd_0_0`。

![RDD的Partition与Block的逻辑关系](../images/2018/12/rdd的partition与block的逻辑关系.png)

### 存储模块架构
`BlockManager`是Storage模块与其他模块交互的最主要的类，它提供了读写Block的接口，并在内部完成对Block的管理。Storage模块采用Master/Slave架构，主从架构就体现在`BlockManager`上。`BlockManager`运行在Driver和每个Executor上:
- 运行在Driver上的`BlockManager`是Master，负责整个Application的Block的元数据信息的维护和管理；
- 运行在Executor上的`BlockManager`是Slave，负责管理该Executor上Block的信息，将信息汇报到位于Driver端的`BlockManager`，并接收并执行来自Driver端`BlockManager`的命令

如下图：

![Storage模块](https://upload-images.jianshu.io/upload_images/35301-d83f9c38f26f5752.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

### BlockManager之间的交互(主从架构)
BlockManager的主从节点之间完成通信具体结构如图所示。

![BlockManager的主从架构](/images/2019/06/blockmanager的主从架构.png)

`BlockManager`的主从节点之间完成远程通信具体是通过几个成员变量来完成的。

- BlockManagerMasterEndpoint
  只存在于Master节点上，由Driver上的SparkEnv负责创建和注册到Driver的RpcEnv中。负责管理所有节点上的BlockManager，包括BlockManager与Executor的映射关系，以及Block的位置信息(即Block所在的BlockManager)等。
  Driver或Executor上`BlockManagerMaster`的`driverEndpoint`属性将持有`BlockManagerMasterEndpoint`的`RpcEndpointRef`，用于RPC通信。
- BlockManagerSlaveEndpoint
  每个Executor和Driver都有属于自己的BlockManagerSlaveEndpoint，分别由各自的SparkEnv负责创建和注册到各自的RpcEnv中。
  `BlockManagerSlaveEndpoint`接收来自于`BlockManagerMasterEndpoint`发来的命令。`BlockManagerSlaveEndpoint`在`BlockManager`中包裹在属性`slaveEndpoint`中
  在向Driver注册BlockManager时，会带上`BlockManagerSlaveEndpoint`的信息：
  ```Scala
  val idFromMaster = master.registerBlockManager(id, maxOnHeapMemory, maxOffHeapMemory, slaveEndpoint)
  ```
- BlockManagerMaster
  代理BlockManager与Driver上的BlockManagerMasterEndpoint通信。其本身并不是分布式的Master，而只是用于与Master进行通信。Driver和Executor上都会有一个BlockManagerMaster。Driver中的BlockManagerMaster保存有BlockManagerMasterEndpoint的实例，Executor中的BlockManagerMaster保存有实例的引用，用于发起RPC调用。

在Driver和每个Executor上都会有一个运行的BlockManager实例。
BlockManager会在每个节点上(driver和executor)都运行一个实例，用于提供block管理的接口。BlockManager的几个成员作用：

- ShuffleClient
  读取其他Executor shuffle output的客户端。如果开启了外部混洗服务，会在每个节点上运行一个独立于Executor之外的长期运行的服务，此时需要另创建ShuffleClient。如果没有开启外部混洗服务，直接使用初始化后的BlockTransferService作为ShuffleClient
- BlockManagerMaster
  统一管理位于Executor上的BlockManager
- DiskBlockManager
  磁盘块管理
- MemoryStore
  内存管理
- DiskStore

BlockManager在使用前，首先需要调用它的`initialize`方法。`initialize`方法主要完成以下几个内容：
1. BlockTransferService的初始化和ShuffleClient的初始化。
    - BlockTransferService的初始化需要提供BlockDataManager对象，用于获取或存放数据块
    - ShuffleClient是读取其他Executor输出文件的客户端。根据是否开启外部混洗服务，而采用不同的初始化。外部混洗服务是运行在每个节点上的、独立与Executor之外的一个长期运行的服务。这样，在没有开启外部混洗服务时，每个Executor会充当shuffle file的server。在开启了外部混洗服务后，会使用这个额外运行的服务作为shuffle file的server，原本的Executor可以被提前回收掉。外部混洗服务通过配置`spark.shuffle.service.enable`开启。如果没有开启，ShuffleClient就是前面初始化过的blockTransferService
2. 创建BlockManagerId
    BlockManagerId是BlockManager的唯一身份标志
3. 向Master注册该BlockManager
4. 初始化ShuffleServerId
  ShuffleServer用于充当读取Shuffle file的服务器。如果没有开启external shuffle service，那么就直接使用BlockManagerId
5. 如果开启了external shuffle service，并且当前节点不是Driver，还需要向Shuffle Server进行注册。


### ShuffleClient
ShuffleClient既充当客户端，也充当Server：
- 访问远端节点的Executor的BLockManager的ShuffleClient中的TransportServer提供的RPC服务，下载或上传Block
- 运行TransportServer，供远端节点访问，上传或下载Block

如果开启了外部混洗服务，会建立`ExternalShuffleClient`；否则会直接使用BlockManager构造函数中提供的`BlockTransferService`。




### BlockManagerMaster对BlockManager的管理



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
