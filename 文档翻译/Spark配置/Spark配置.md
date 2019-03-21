[原文链接](https://spark.apache.org/docs/latest/configuration.html#viewing-spark-properties)

# Spark配置
Spark提供了一下三处配置系统的地方：
- Spark属性(Spark properties)控制绝大部分参数，可以通过SparkConf对象设置，也可以通过Java系统属性来设置(集群中的机器统一配置)
- 环境变量，配置集群中的单个机器，例如IP地址。通过在集群每个节点上Spark安装目录下的`spark-env.sh`脚本来配置
- 日志，通过`log4j.properties`文件配置


## Spark属性
Spark属性控制应用程序的大部分配置，每个应用程序可以有不同的配置。这些属性可以直接在`SparkConf`对象上设置，然后用作创建`SparkContext`对象的参数。在`SparkConf`对象上，可以设置一些常用的属性(如Master的URL，Application的名称等)，也可以使用`set()`方法设置任意的key-value对属性。如下面的例子，初始化一个在本地运行的使用两个线程的应用程序：
```
val conf = new SparkConf()
             .setMaster("local[2]")
             .setAppName("CountingSheep")
val sc = new SparkContext(conf)
```

其中，`local[2]`表示两个线程——最小的并发方式，用于帮助我们找到只有在分布式环境下才会出现的问题。Local模式下可以启用1个或以上的线程，但是对于Streaming程序中，由于需要有接收线程和处理线程，因此至少要启动两个线程。

配置时间长度的属性应指明长度单位，下面这些形式都是可以接受的：
```
25ms (milliseconds)
5s (seconds)
10m or 10min (minutes)
3h (hours)
5d (days)
1y (years)
```
同样，指定大小的属性也需要指定单位：
```
1b (bytes)
1k or 1kb (kibibytes = 1024 bytes)
1m or 1mb (mebibytes = 1024 kibibytes)
1g or 1gb (gibibytes = 1024 mebibytes)
1t or 1tb (tebibytes = 1024 gibibytes)
1p or 1pb (pebibytes = 1024 tebibytes)
```
不指定单位时，大部分情况下会默认单位为byte，但并不是全部，有的地方会解释为KB或MB。

### 动态加载Spark配置
一些情况下，可能需要避免将配置硬编码到代码中。比如，想要以不同的master或不同大小的内存运行同样的代码。这种情况，可以创建一个空的配置：
```
val sc = new SparkContext(new SparkConf())
```
然后在提交Spark程序时指定配置参数：
```
./bin/spark-submit --name "My app" --master local[4] --conf spark.eventLog.enabled=false
  --conf "spark.executor.extraJavaOptions=-XX:+PrintGCDetails -XX:+PrintGCTimeStamps" myApp.jar
```

Spark Shell和`spark-submit`工具支持两种方式动态加载配置：
- 命令行选项
  例如上面的`--master`选项。`spark-submit`可以使用`--conf`选项配置来配置任意的属性，不过对于一些常用的选项，工具提供了专门的选项，比如`--master`、`--name`等。
- 从`spark-defaults.conf`中读取配置
  文件中的每一行都包含了一个key和对应的value，如：
  ```
  spark.master            spark://5.6.7.8:7077
  spark.executor.memory   4g
  spark.eventLog.enabled  true
  spark.serializer        org.apache.spark.serializer.KryoSerializer
  ```

无论用何种方式(命令行选项还是配置文件)，最终这些配置都会与代码中指定的`SparkConf`的配置进行合并。这三种配置来源的优先级为：
1. 在代码中`SparkConf`对象上直接设置的优先级最高
2. 命令行选项优先级其次
3. `spark-defaults.conf`配置文件中的优先级最低

Spark早期版本的某些配置项的关键字已经被移除，但这些配置仍然会被接受，只不过会比新的配置关键字的优先级低。

Spark属性主要可以分为两大类：
- 配置属性
  这种属性要么在程序运行起来后，通过在`SparkConf`中配置就已经无效了，要么行为要依赖于使用的资源管理器和选择的模式。这类属性包括如`spark.driver.memory`、`spark.executor.instances`这些，建议使用命令行选项或配置文件的方式来配置
- Spark运行时行为控制
  比如`spark.task.maxFailures`这类属性，可以通过任意一种方式配置

### 查看Spark属性
Spark应用程序的web UI在'Environment'选项卡中列出了Spark的属性。这里可以用于检查配置是否生效，以及是否正确。要注意的一点是，只有在配置文件、SparkConf对象上或命令行选项中显式指定的选项才会出现在这里。其他的配置选项，都使用默认的配置，一般不出现在web UI上。

### 常用配置选项
[配置选项](https://spark.apache.org/docs/latest/configuration.html#viewing-spark-properties)

### 资源管理器
Spark使用的资源管理器，根据类型的不同，还有进一步的配置

[YARN](https://spark.apache.org/docs/latest/running-on-yarn.html#configuration)

## 环境变量
## 日志配置
