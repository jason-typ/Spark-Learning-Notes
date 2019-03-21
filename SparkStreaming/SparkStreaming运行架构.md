> 《图解Spark核心技术与案例实战》

## SparkStreaming运行时架构
使用SparkStreaming对流数据进行处理大致可以分为4个步骤：
- 启动流处理引擎
- 接收及存储流数据
- 处理流数据
- 输出处理结果

其运行架构如下图所示：

![SparkStreaming运行时架构](/images/2019/03/sparkstreaming运行架构.png)

SparkStreaming主要有两个功能：
- 流数据的接收与存储
- 批处理作业的生成与管理

SparkStreaming分为Driver端和Client端：
- Driver端
  运行的是StreamingContext实例，该实例包括DStreamGraph和JobScheduler(包括ReceiverTracker和JobGenerator)
- Client端
  接收数据的组件包括Receiver和ReceiverSupervisor

数据处理的流程分为以下：
1. 初始化`StreamingContext`对象。该对象初始化过程中会实例化`DStreamGraph`和`JobScheduler`对象。
    - `DStreamGraph`，用于存放DStream以及DStream之间的依赖关系等信息。
    - `JobScheduler`，包含`ReceiverTracker`和`JobGenerator`
      - `ReceiverTracker`，数据接收器(Receiver)的管理者，负责启动、管理各个Receiver，以及Receiver接收到的数据。`ReceiverTracker`在启动过程中，会在Executor中启动数据接收管理器(`ReceiverSupervisor`)，再由`ReceiverSupervisor`启动数据接收器Receiver
      - `JobGenerator`，批处理作业生成器
2. 数据接收器Receiver启动，接收实时数据。Receiver会将接收到的数据交给ReceiverSupervisor，由其进行数据转储操作，将数据存放到Worker的内存或磁盘上。数据存储结束后，ReceiverSupervisor会将数据存储的元信息上报给ReceiverTracker。在ReceiverTracker中有一个ReceivedBlockTracker成员，它会负责管理数据块的元信息
3. JobGenerator中维护一个定时器，周期的生成微批处理作业
4. 数据处理完毕后，输出到外部系统
