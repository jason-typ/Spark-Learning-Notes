# Spark术语

- Application：用户编写的Spark应用程序
- Driver：用户提交的应用程序，运行起来后就是Driver。Driver中必须要有一个SparkContext(运行时环境)。
- SparkContext：Spark程序运行时环境，负责与集群资源管理器和Executor通信交互
- Executor：运行在WorkerNode上的一个进程(每个Application都会在自己的WorkNode上启动这么一个专属的进程)，负责将task包装成taskRunner，并从线程池中取出一个空闲线程来运行这个任务
- cluster manager：资源管理器，Spark支持的有：
  - standalone，Spark自带的一个资源管理器
  - Yarn
  - Mesos
-
