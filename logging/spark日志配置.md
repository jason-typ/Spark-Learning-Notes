### 1. spark job在YARN上的日志收集
在YARN上，Spark Application的Driver和Executors都运行在Container中。对Containers中产生的日志，YARN有两种处理方式：

1. 本地模式
默认情况下，日志会保存在各个机器的本地文件系统上。YARN配置`yarn.nodemanager.log-dirs`指定了container产生的日志的存放位置。本地文件系统上的日志在指定时间间隔后会被删除。有两种查看日志方式：
    1. 登录到各台机器上去查看日志，这些日志会按应用id和容器id的结构来分子目录保存
      ```
      [root@ip-10-201-1-114 containers]# pwd
      /mnt/var/log/hadoop-yarn/containers
      [root@ip-10-201-1-114 containers]# tree
      .
      ├── application_1541141505942_0075
      │   └── container_1541141505942_0075_02_000002
      │       ├── stderr.gz
      │       └── stdout.gz
      └── application_1541141505942_0077
          ├── container_1541141505942_0077_01_000003
          │   ├── stderr
          │   └── stdout
          └── container_1541141505942_0077_01_000009
              ├── stderr
              └── stdout
      ```
    2. 通过Spark Web Ui来查看日志，而不需要MapReduce历史服务器
2. 归集模式
如果开启了日志归集(通过设置`yarn.log-aggregation-enable`为true设置)，在应用程序完成后(所以运行中的程序日志在hdfs上是看不到的)，log会被拷贝到HDFS上，本地文件将被删除。这些日志可以通过三种方式查看：
    1. `yarn logs -applicationId id`命令在集群的任何节点查看，这条命令会打印出应用所有容器的日志文件内容。
    2. 通过HDFS查看，日志存放目录通过`yarn.nodemanager.remote-app-log-dir`和`yarn.nodemanager.remote-app-log-dir-suffix`进行配置。
    3. 通过Spark Web UI(Executors tab)查看。前提是需要在 yarn-site.xml中配置 yarn.log.server.url，并保证Spark历史服务器和MapReduce历史服务器正常运行。Spark历史服务器UI上的log连接会重定向到MapReduce历史服务器来显示归集的日志

### 2. Spark On YARN定制化log4j
Spark使用log4j记录日志，配置文件为`spark/conf/log4j.properties`。有几种可选方式给Driver和Executor配置定制化的log4j配置：
1. 在使用`spark-submit`提交Spark应用程序时，通过` --files`选项来上传一个配置过的log4j.properties文件
2. 通过`--config`选项，在执行器的`spark.driver.extraJavaOptions`，或在驱动器的`spark.executor.extraJavaOptions`选项，增加`-Dlog4j.configuration=<location of configuration file>`配置。必须使用`file:`协议，并且指定的文件在每个节点上都必须存在
3. 更新$SPARK_CONF_DIR/log4j.properties文件，此配置会和其他配置一起自动上传。注：如果指定了多个选项，另外2个选项的优先级高于本配置

如果采用第一个方式，需要注意的是，执行器和应用Master使用同一个log4j配置，当他们跑在同一个节点的时候会有问题（如，尝试写同一个日志文件)。

还要注意第二个方法使用cluster方式提交spark Application时，在client端(命令行)会接受一些简单的信息打印出来，也就是说仍会用到Sparkconf目录下的log4j.properties，注意这个文件的配置。

### 3. Spark Job配置定制的log4j方法
基于2中的介绍，可以在提交Spark应用程序时指定特定的配置过的log4j文件：使用`--file`上传文件，然后搭配-`-config`选项指定文件。命令示例为：

```
spark-submit --file /usr/local/spark/conf/log4j-executor.properties,/usr/local/spark/conf/log4j-driver.properties --conf spark.driver.extraJavaOptions="-Dlog4j.configuration=log4j-driver.properties" --conf spark.executor.extraJavaOptions="-Dlog4j.configuration=log4j-executor.properties --master yarn --class com.strikingly.dime.spark.DimeJob --deploy-mode cluster ./dime-jobs.jar"
```

其中`spark.driver.extraJavaOptions`可以使用`--driver-java-options`替换掉，即：

```
spark-submit --file /usr/local/spark/conf/log4j-executor.properties,/usr/local/spark/conf/log4j-driver.properties --driver-java-options "-Dlog4j.configuration=log4j-driver.properties" --conf spark.executor.extraJavaOptions="-Dlog4j.configuration=log4j-executor.properties --master yarn --class com.strikingly.dime.spark.DimeJob --deploy-mode cluster ./dime-jobs.jar"
```

这样为Driver和Executor分别指定了配置过的log4j配置文件。当然，如果能够确保每个机器上面都能找到一份log4j配置，就不需要使用`--file`选项上传了，只要指定本机上配置文件的位置就可以，如：

```
spark-submit --file /usr/local/spark/conf/log4j-executor.properties --driver-java-options "-Dlog4j.configuration=file:/usr/local/spark/conf/log4j-driver.properties" --conf spark.executor.extraJavaOptions="-Dlog4j.configuration=log4j-executor.properties --master yarn --class com.strikingly.dime.spark.DimeJob --deploy-mode cluster ./dime-jobs.jar"
```

在配置文件中，为了与YARN搭配，使得YARN能够收集Container中的日志，log4j的文件路径配置为：

```
log4j.appender.DayRoll.File=${spark.yarn.app.container.log.dir}/stderr
```
这里的`${spark.yarn.app.container.log.dir}`是YARN存放Container日志的位置。要注意的是，在cluster模式下，除了Driver和Executor，还有client端(提交命令的命令行)也会打印log信息。但是在client端是没有`${spark.yarn.app.container.log.dir}`这个值的。因此如果client端的log4j配置(即默认的log4j.properties文件)也是这样配置，那么client端的文件路径其实是`/stderr`，而通常对这个文件是没有权限创建的，因此任务会提交成功，但是命令行上显示失败，原因就是这个。
