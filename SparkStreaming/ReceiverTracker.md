[原文链接](https://www.jianshu.com/p/3195fb3c4191)

## ReceiverTracker
`ReceiverTracker`，它是 `SparkStream`作业调度器 `JobScheduler`的成员，是数据接收器Receiver的管理者，负责启动、管理各个receiver，以及各个receiver接收到的数据。其运行时架构如图。

![SparkStreaming运行时架构](/images/2019/03/sparkstreaming运行架构.png)

大致的工作流程为：
1. 启动
  ReceiverTracker在启动过程中，会在Executor中启动数据接收管理器(ReceiverSupervisor)，再由ReceiverSupervisor启动数据接收器Receiver
2. 数据接收。Receiver启动后开始接收实时数据
  Receiver会将接收到的数据交给ReceiverSupervisor，由其进行数据转储操作，将数据存放到Worker的内存或磁盘上。数据存储结束后，ReceiverSupervisor会将数据存储的元信息上报给ReceiverTracker。在ReceiverTracker中有一个ReceivedBlockTracker成员，它会负责管理数据块的元信息

下面结合代码详细了解ReceiverTracker。

## Receiver启动
![ReceiverTracker启动](/images/2019/03/receivertracker启动.png)

Receivers由ReceiverTracker管理，Receiver的启动从ReceiverTracker对象的创建开始。从StreamingContext的start方法开始，会调用到`JobScheduler.start`方法。在`JobScheduler.start`方法中，会创建一个`ReceiverTracker`的实例，并调用`ReceiverTracker.start`方法。

### `ReceiverTracker.start`方法
![ReceiverTracker#start方法](/images/2019/03/receivertracker-start方法.png)
这个方法中主要完成两件事情：
1. 在内部初始化一个`endpoint: ReceiverTrackerEndpoint`实例对象。这个endpoint是一个注册的RPC调用，从前面的图中可以看到，ReceiverTracker和Receiver分别位于Driver端和Executor端，endpoint可以作为一个桥梁，接收、处理来自ReceiverTracker以及Receiver的消息。
2. 调用`launchReceivers`方法，将Receivers分发到各个工作节点(Executors)上，并运行这些Receivers

### `ReceiverTracker.launchReceivers`方法
![ReceiverTracker-launchReceivers方法](/images/2019/03/receivertracker-launchreceivers方法.png)
`launchReceivers`方法负责分发Receivers到各个Executors中，然后运行这些Receivers。具体是通过两部完成的：
1. 获取所有的Receivers
  通过`ReceiverInputDStream`的`getReceiver`接口获取
2. 给endpoint发送`StartAllReceivers`消息
  前面介绍endpoint是Driver端和Executor端的一个桥梁，负责两端的通信与请求

### ReceiverTrackerEndpoint处理`StartAllReceivers`消息
![处理StartAllReceivers消息](images/2019/03/处理startallreceivers消息.png)

`ReceiverTrackerEndpoint`在收到`StartAllReceivers`消息后，会首先将Receivers分配到Executors中。分配的步骤如下：
- 将带有prefered location(host)的receivers，尽量均匀的分配到该host上的executors上
- 将剩余的receivers均匀的分布到所有的executors中，来尽量让整体均匀分布

分配的结果以map的形式返回，其中key是receiver id，value是该receiver id对应的目的executor列表。确定了Receivers分配到哪些Executors中启动后，下一步就是通过调用`startReceiver()`真正启动Receivers。

### ReceiverTrackerEndpoint.startReceiver方法
![启动Receivers](images/2019/03/启动receivers.png)

这里具体完成Receiver的启动。首先将Receiver待要启动的位置列表封装成RDD格式`receiverRDD`，然后定义了对`receiverRDD`(启动Receiver的位置，可以是host，也可以是host+executorID提供的更准确的位置)的操作`startReceiverFunc`。然后作为一个job提交运行，代码如下
```
val future = ssc.sparkContext.submitJob[Receiver[_], Unit, Unit](
        receiverRDD, startReceiverFunc, Seq(0), (_, _) => Unit, ())
```

在`startReceiverFunc`中定义了在Executor端的动作，代码如下：
```
val supervisor = new ReceiverSupervisorImpl(
    receiver, SparkEnv.get, serializableHadoopConf.value, checkpointDirOption)
  supervisor.start()
  supervisor.awaitTermination()
```
在Executor中启动了一个`ReceiverSupervisor`实例，并调用`start`方法。`start`方法中首先会启动`BlockGenerator`，用于将Receiver收到的数据变为数据块；然后会到Driver端注册这个Receiver。涉及到的代码如下
```
override protected def onReceiverStart(): Boolean = {
  val msg = RegisterReceiver(
    streamId, receiver.getClass.getSimpleName, host, executorId, endpoint)
  trackerEndpoint.askSync[Boolean](msg)
}
```
即向`ReceiverTrackerEndpoint`发送`RegisterReceiver`消息。最后，会调用到具体的Receiver自己的start方法。这部分就取决于具体的Receiver的实现了。

supervisor实例在一个线程中启动后，会调用`supervisor.awaitTermination`阻塞住，让其一直运行

## Receiver接收数据流程
Receiver接收到数据后的流转图如下：
![数据流转](/images/2019/03/数据流转.png)

### 1. Receiver -> ReceiverSupervisor
Receiver调用`store`方法，将接收到的数据传给ReceiverSupervisor。`store`方法根据参数不同，有多个不同的重载版本，用于存放不同格式的数据

### 2. ReceiverSupervisor将数据存放到内存/硬盘，并上报
`store`方法会进一步调用ReceiverSupervisor的`push**`方法，会将收到的数据以数据块的形式存储。`push`方法也根据`store`方法的不同有不同的版本，用于存放不同形式的数据。如
- pushSingle，对应单条小数据
- pushArrayBuffer，对应数组形式数据
无论是哪中形式数据对应的哪个具体的方法，最终都会调用到`ReceiverSupervisor.pushAndReportBlock`这个方法，用于保存data block到内存/磁盘，并上报给Driver。

#### 2.1 数据保存到内存/磁盘
涉及到的代码在`pushAndReportBlock`方法中：
```
val blockStoreResult = receivedBlockHandler.storeBlock(blockId, receivedBlock)
```
得到保存数据的结果。其中的`ReceivedBlockHandler: ReceivedBlockHandler`会根据是否开启了WAL(配置参数`spark.streaming.receiver.writeAheadLog.enable`，默认关闭)决定实例化的类型：`WriteAheadLogBasedBlockHandler`和`BlockManagerBasedBlockHandler`。以`BlockManagerBasedBlockHandler`为例，调用它的`storeBlock`方法，会根据数据形式的不同稍有不同。但最终都会调用到`BlockManager`的`put**`方法，由`BlockManager`完成将数据库存放到内存/磁盘的工作。

所以，这部分的数据流向是ReceiverSupervisor -> BlockManager -> memory/disk

#### 2.2 上报元信息
数据块保存后，得到保存结果，并封装得到一个block描述信息`blockInfo: ReceivedBlockInfo`，这里面包含streamId、数据位置、数据条数、数据 size 等信息。然后以block信息为参数，向ReceiverTracker(ReceiverTrackerEndpoint)发送`AddBlock`消息。
这一部分是元信息的流转，从ReceiverSupervisor -> ReceiverTracker(Endpoint)

#### 3. ReceiverTracker处理AddBlock消息
ReceiverTrackerEndpoint收到AddBlock消息后，会调用`addBlock`方法，并返回结果。在`ReceiverTrackerEndpoint.addBlock`方法中又会调用到`ReceivedBlockTracker.addBlock`方法。在这个`addBlock`方法中，首先会根据是否配置有CheckPoint Dir，决定是否需要将信息先写到WAL中，然后会将元数据信息存放到`streamIdToUnallocatedBlockQueues: mutable.HashMap[Int, ReceivedBlockQueue]`中。这是一个HashMap，其中key为streamId，value为从该Stream中接收到的Block的Queue。
```
streamIdToUnallocatedBlockQueues.getOrElseUpdate(streamId, new ReceivedBlockQueue)
```
所有的Block信息都是由BlockManager管理的，至此，就完成了数据本身的存储，以及数据元信息的上报。
