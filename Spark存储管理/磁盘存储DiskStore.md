## 磁盘存储DiskStore
当MemoryStore没有足够空间时，或者，当手动的设置RDD的存储级别为磁盘时，就会使用DiskStore将块存入磁盘。

```Scala
class DiskStore(
      conf: SparkConf,
      diskManager: DiskBlockManager,
      securityManager: SecurityManager)
```

`DiskStore`依赖于定义中的`DiskBlockManager`进行文件的管理(Block ID与磁盘上文件的映射关系)，如初始化目录、获取文件等操作。具体的读、写操作是直接在`DiskStore`中直接完成的。

### 1. blockSizes
`DiskStore`内部维护了一个名为`blockSizes`的并发哈希表，用来保存blockId与存放的block的大小之间的映射关系

### 2. 从磁盘读取block数据
使用`getBytes()`方法，返回一个`DiskBlockData`类型的对象。
```Scala
private class DiskBlockData(
    minMemoryMapBytes: Long,
    maxMemoryMapBytes: Long,
    file: File,
    blockSize: Long) extends BlockData
```
### 3. 往磁盘中写数据
使用`putBytes()`、`put`方法

## 磁盘块管理DiskBlockManager
一个逻辑上的Block会根据BlockId生成一个文件名，并映射到物理设备上的文件，存放到`spark.local.dir`设置的目录下面。

### 初始化
BlockManager初始化时会创建DiskBlockManager。DiskBlockManager的构造步骤为：

1. 创建本地文件目录localDirs。调用`createLocalDirs`方法，根据Spark配置中的`spark.local.dir`，为每一项都创建一个本地目录
2. 创建localDirs下面的子目录，用二维数组`subDirs`来表示。一级目录已经在conf文件中通过配置`spark.local.dir`指定了，因此不可修改。二级目录只是指定了数量，对应数组的大小。因此注释中写着subDirs不可变，但是subDirs(i)是可变的。
3. 添加运行结束时钩子。使用`DiskBlockManager`的`doStop`方法，删除`localDirs`以及其下的子目录。

### 获取磁盘文件

获取磁盘文件使用`getFile`方法` def getFile(blockId: BlockId): File`。处理步骤为：
1. 根据文件名计算哈希值
2. 根据哈希值确定一级目录dirId：目录确定一个圈，哈希值对目录总数取余
3. 确定二级目录subDirId：哈希值对一级目录总数求商，再对二级目录总数取余
4. 如果dirId/subDirId目录存在，则获取该目录下的文件，否则新建dirId/subDirId目录

### createTempShuffleBlock
当需要保存shuffle过程的临时输出时，会调用到这个方法创建一个临时的、唯一的blockId。

```Scala
def createTempShuffleBlock(): (TempShuffleBlockId, File) = {
    var blockId = new TempShuffleBlockId(UUID.randomUUID())
    while (getFile(blockId).exists()) {
      blockId = new TempShuffleBlockId(UUID.randomUUID())
    }
    (blockId, getFile(blockId))
  }
```
TempShuffleBlockId的生成规则：temp_shuffle_后面加上UUID。方法中会判断是否已经存在这个文件，存在的话就生成新的UUID，直到这个文件不存在，以此来保证唯一性。
