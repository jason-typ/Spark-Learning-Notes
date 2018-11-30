# Spark配置

除了在代码层面完成对SparkConf对象的属性的配置。还可以直接在提交应用程序(spark-submit)时，通过指定参数来完成这一工作。这些配置会被当成环境变量，完成SparkContext的初始化。

Spark为一些常用的配置设置了可配置参数：

- --master，表示要连接的资源管理器
- --deploy-mode，取值client或cluster。表示在本地还是在cluster的一个工作节点上启动driver。默认是client
- --class，main函数，程序入口
- --name，该Application的名称，如“WordCount”
- --jars，Application需要依赖的jar包列表，以逗号分隔。jar包会被上传到集群中
- --files，以逗号分隔的文件列表，会被放到每个Executor的工作目录中。每个Executor可以直接使用`SparkFiles.get(fileName)`来访问
- --executor-memory，每个Executor可以使用的内存，(如1000M, 2G)，默认1G
- --driver-memory，Driver进程可以使用的内存量(如1000M，2G)，默认1024M

另外，还可以通过`--config prop=value`来进行任意的属性配置，如`--config spark.executor.memory=1G`(这等同于`--executor-memory`)。

也可以通过`--properties-file FILE`指定包含键值对的属性配置文件来简化命令的输入，或多环境的情况。默认情况下，spark-submit脚本会在Spark的安装目录下找到` conf/spark-defaults.conf`文件，读取文件中的属性配置。一般会使用配置文件的方式，在启动任务时指定配置文件的位置。

```
./spark-submit --class com.data.SparkLearning --properties-file my-config.conf
```

对于在多个位置进行的相同配置，Spark有一个优先级来确定最终的配置，以下按照优先级从高到低排序：

1. 用户代码中使用set方法的配置
2. 通过spark-submit传递的参数
3. 配置文件中的配置
4. 默认值

其他一些需要注意的Spark的配置项如下：

参数  |  描述
--|--
spark.executor.cores</br>(--executor-cores)  |
spark.speculation  |
  |

参数  | 默认值  |  描述
--|---|--
spark.executor.cores</br>(--executor-cores)  |   |  每个Executor可以使用的核数。在YARN中默认是1，在standalone模式中，默认是主机的核数
spark.speculation  | false  |  预测执行机制。打开后，当某个任务执行较慢时，会在另外的节点上启动这个任务的副本
spark.default.parallelism  | 对于分布式的混洗操作，默认为父RDD中最大的分区数。没有父RDD，取决于部署模式：Local，本机的核数；Mesos，8；其他，所有Executor机器的核数与2中的较大值  |RDD的默认分区数
  |   |
  |   |
  |   |
  |   |
  |   |
  |   |
