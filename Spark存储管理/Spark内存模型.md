[Spark内存管理详解(上)](https://www.jianshu.com/p/3981b14df76b)

[Spark内存管理详解(下)](https://www.jianshu.com/p/58288b862030)

[Spark Block存储管理](http://shiyanjun.cn/archives/1641.html)

[Apache Spark 统一内存管理模型详解](https://www.iteblog.com/archives/2342.html#Off-heap_Memory)
[Spark内存占用计算](https://blog.csdn.net/lingbo229/article/details/80914283)


Spark存储结构分为两层：内存存储和磁盘存储。磁盘存储用于在内存不够时进行辅助存储。

Spark的内存分为两部分：堆内内存和堆外内存。

### 1. Spark内存管理
Spark目前有两种内存管理模型：静态内存管理和统一内存管理。在`SparkEnv`中可以看到初始化：

```
val useLegacyMemoryManager = conf.getBoolean("spark.memory.useLegacyMode", false)
val memoryManager: MemoryManager =
  if (useLegacyMemoryManager) {
    new StaticMemoryManager(conf, numUsableCores)
  } else {
    UnifiedMemoryManager(conf, numUsableCores)
  }
```

Spark1.6之后默认使用统一内存管理模型，如果要使用静态内存管理，需要配置`spark.memory.useLegacyMode`为true。这两种模型区别在于对空间分配的方式。

下面主要介绍的是统一内存管理模型。

#### 1.1 堆内内存与堆外内存
作为一个JVM进程，Executor的内存管理建立在JVM的内存管理之上。Spark对JVM的堆内(On-Heap)空间进行了更为详细的分配，以充分利用内存。同时，Spark引入了堆外(Off-Heap)内存，使之可以直接在工作节点的系统内存中开辟空间，进一步优化了内存的使用。

![堆内和堆外内存](https://upload-images.jianshu.io/upload_images/35301-c896dd56ffe88b27.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/770/format/webp)

#### 1.1.1 堆内内存
堆内内存的大小，由Spark应用程序启动时的`-executor-memory`或配置参数`spark.executor.memory`指定。Executor内运行的并发任务共享JVM堆内内存。

堆内内存大致可以分为三个部分：
- **存储内存**，任务在缓存RDD和广播(Broadcast)数据时占用的内存被规划为存储(Storage)内存
- **执行内存**，在执行Shuffle、Join、Sort等计算过程中占用的内存被规划为执行内存
- **其他**，剩余部分不做规划。哪些Spark内部的对象实例，或者用户定义的Spark应用程序中的对象实例，均占用剩余的空间。

不同的内存管理模式下，这三部分占用的空间大小各不相同。

#### 1.1.2 堆外内存
为了进一步优化内存的使用以及提高Shuffle时排序的效率，Spark引入了堆外(Off-Heap)内存，试制可以直接在工作节点的系统内存中开辟空间，存储经过序列化的二进制数据。

默认情况下堆外内存并不启用，可以通过配置`spark.memory.offHeap.enabled`参数启用，并由`spark.memory.offHeap.size`参数设定堆外空间的大小。

除了没有other空间，堆外内存与堆内内存的划分方式相同，**所有**运行中的并发任务共享(堆外)存储内存和执行内存。

#### 1.1.3 接口
Spark为存储内存和执行内存的管理提供了统一的接口：`MemoryManager`。同一个`Executor`内的任务都调用这个接口的方法来申请或释放内存，同时在调用这些方法时都需要指定内存模式(MemoryMode)，这个参数决定了是在堆内还是堆外完成这次操作。

`MemoryManager`的具体实现上，Spark1.6之后默认使用同一内存管理模型(Unified Memory Manager)，1.6之前的静态管理(Static Memory Manager)方式仍然保留，可以通过配置参数`spark.memory.useLegacyMode`启用。

具体代码可以查看`SparkEnv`中memoryManager的创建部分：
```
val useLegacyMemoryManager = conf.getBoolean("spark.memory.useLegacyMode", false)
val memoryManager: MemoryManager =
  if (useLegacyMemoryManager) {
    new StaticMemoryManager(conf, numUsableCores)
  } else {
    UnifiedMemoryManager(conf, numUsableCores)
  }
```
### 2. 内存空间分配
不同的管理模式下，具有不同的内存空间分配

#### 2.1 静态内存管理

##### 2.1.1 静态内存管理的堆内内存
在静态内存管理机制下，**存储内存**、**执行内存**和**其他内存**三部分的大小在Spark应用程序运行期间是固定的，但用户可以在应用程序启动前进行配置。堆内内存的分配如图所示：

![静态内存管理的堆内内存分配](https://upload-images.jianshu.io/upload_images/35301-1c5ca5f6aa6cbd28.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

- **系统总内存**
  Executor所使用的Java堆内存的总大小，`systemMaxMemory`，取决于当前JVM堆内内存的大小。可以在Spark应用程序启动时，通过参数`spark.executor.memory`指定
- **Storage内存区域**
  Storage内存区域占用系统可用内存的固定大小，由配置参数`spark.storage.memoryFraction`控制，默认为0.6，即占系统可用内存的60%。Storage内存区域中又包含两部分：
  - **预留部分**，用于防止OOM，使用配置参数1 - `spark.storage.safetyFraction`控制
  - **可用Storage部分**，用于缓存RDD数据和broadcast数据。由参数`spark.storage.safetyFraction`控制。默认是0.9，即占总Storage内存区域的90%
- **Execution内存区域**
  Execution内存区域也占系统可用内存的固定大小，由参数`spark.shuffle.memoryFraction`控制，默认为0.2，即占系统可用内存的20%。Execution内存区域同样包含两部分：
  - **可用Execution内存区域**，用于缓存shuffle过程中的中间数据。由参数`spark.shuffle.safetyFraction`控制，默认0.8，即占总Execution内存区域的80%
  - **预留部分**，用于防止OOM，包括剩下的Execution内存区域，即(1 - `spark.shuffle.safetyFraction`)*Execution总内存区域
- **Other部分**
  用于存放用户定义的数据结构或Spark内部元数据等。大小为剩余系统可用内存，即`systemMaxMemory - Storage - Executor`

按照以上介绍，Executor中可用的堆内内存：
- `可用存储内存 = systemMaxMemory * spark.storage.memoryFraction * spark.storage.safetyFraction`
- `可用执行内存 = systemMaxMemory * spark.shuffle.memoryFraction * spark.shuffle.safetyFraction`

##### 2.1.2 静态内存管理的堆外内存
堆外内存只有**存储内存**和**执行内存**两部分，大小同样是固定的，如下图：

![静态内存管理的堆外内存分配](https://upload-images.jianshu.io/upload_images/35301-f9f9801130508114.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

堆外内存总大小由参数`spark.memory.offHeap.size`控制。可用的存储内存大小由参数`spark.memory.storageFraction`指定占总堆外内存大小的百分比。默认50%。堆外内存的剩余部分全是堆外执行内存部分。

##### 2.1.3 静态内存分配局限
静态内存管理机制实现起来较简单，但如果配置不当，很容易造成存储内存和执行内存中的一方内存资源紧张，而另一方仍有大量剩余空间。出于对旧版本的兼容，目前仍然保留了这种内存管理方式。默认是不使用的。

#### 2.2 统一内存管理
Spark1.6之后引入统一内存管理机制，与静态内存管理不同的是：存储内存和执行内存共享同一块空间，并且可以动态占用对方的空闲区域。

##### 2.2.1 统一内存管理的堆内内存
存储内存和执行内存共享同一块空间，并且可以动态占用对方的空闲区域。如下图所示。

![统一内存管理的堆内内存分配](https://upload-images.jianshu.io/upload_images/35301-671d45082ea6ea92.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

- **系统总内存**
  与静态存储模型一样，Executor中可用的总的JVM堆上内存
- **预留内存**
  预留的空间，默认为300M
- **可用内存**
  系统总内存 减去 预留内存。包括两部分，**统一内存**和**其他**
- **统一内存**
  Unified Memory，由Storage和Execution共用，使用参数`spark.memory.fraction`控制，默认为0.6，即占用系统可用内存的60%。统一内存仍然分为Storage和Execution两部分，具体的功能不变。但是可以在己方内存不足时，占用对方的空余空间
  - **Storage内存**
    初始占用统一内存的一定大小，由参数`spark.storage.storage.Fraction`控制
  - **Execution内存**
    占用统一内存剩余的部分
  Storage和Execution都可以在己方内存不足时，占用对方的空余空间。不同的是，Execution在内存不足时，可以强制收回被Storage占用的己方的内存，但反过来不行。
- **其他**
  存放用户定义的数据结构或Spark内部元数据

动态占用的规则如下：
- 设定基本的Storage和Execution内存区域(`spark.storage.storageFraction`参数)，该设定确定了双方各自拥有的空间的范围
- 双方的空间都不足时，则存储到硬盘；若己方空间不足而对方空余时，可借用对方的空间(空间不足时指不足以放下一个完整的Block)
- 执行内存的空间被对方占用后，可以让对方将占用的部分转存到磁盘，然后归还借用的空间
- 存储内存的空间被对方占用后，无法让对方归还。因为执行内存用于存储Shuffle数据，实现比较麻烦

![动态占用机制图解](https://upload-images.jianshu.io/upload_images/35301-ef5c111168db7c31.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)
