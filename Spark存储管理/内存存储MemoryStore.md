
```Scala
class MemoryStore(
    conf: SparkConf,
    blockInfoManager: BlockInfoManager,
    serializerManager: SerializerManager,
    memoryManager: MemoryManager,
    blockEvictionHandler: BlockEvictionHandler)
```

`MemoryStore`负责Block数据在内存中的存储。`MemoryStore`使用一个LinkedHashMap来管理所有的Block：
```Scala
val entries = new LinkedHashMap[BlockId, MemoryEntry[_]](32, 0.75f, true)
private sealed trait MemoryEntry[T] {
  def size: Long
  def memoryMode: MemoryMode
  def classTag: ClassTag[T]
}
```
其中`MemoryMode`有两种：`OnHeap`和`OffHeap`。



###### 1. putIteratorAsValues
以Java变量(反序列化)的方式，直接将Block数据保存在内存中。在内存中对RDD的访问使用迭代器(Iterator)，由于迭代器的所有内容有可能在内存中放不下，从而造成OOM，因此Spark中的Unroll是循序展开的：迭代器读取Record，检查是否有足够空间放得下这条Record，保存Record，然后继续下一条Record。保存的格式是`DeserializedMemoryEntry`。

###### 2. putIteratorAsBytes
同上面的类似，不过保存的格式是二进制格式

###### 3. putBytes
putBytes函数会以序列化的方式保存Block数据。这种方式应该只适用于保存序列化格式的RDD，由于这种RDD可以容易的获取到它的大小。所以可以一次性的申请固定大小的内存空间。

```Scala
def putBytes[T: ClassTag](
      blockId: BlockId,
      size: Long,
      memoryMode: MemoryMode,
      _bytes: () => ChunkedByteBuffer): Boolean = {
    require(!contains(blockId), s"Block $blockId is already present in the MemoryStore")
    if (memoryManager.acquireStorageMemory(blockId, size, memoryMode)) {
      // We acquired enough memory for the block, so go ahead and put it
      val bytes = _bytes()
      assert(bytes.size == size)
      val entry = new SerializedMemoryEntry[T](bytes, memoryMode, implicitly[ClassTag[T]])
      entries.synchronized {
        entries.put(blockId, entry)
      }
      logInfo("Block %s stored as bytes in memory (estimated size %s, free %s)".format(
        blockId, Utils.bytesToString(size), Utils.bytesToString(maxMemory - blocksMemoryUsed)))
      true
    } else {
      false
    }
  }
```
`putBytes`方法中，首先会使用`MemoryManager`来申请Storage内存，保证有足够的空间放置Block。然后创建`SerializedMemoryEntry`(以某种序列化方式保存数据)。最后把(blockId->entry)放到MemoryStore维护的Block的HashMap中。
