## Spark程序在Yarn上的日志管理

## 问题：
1. 如果spark的日志输出放在了多个文件中，yarn使用日志归集后，会不会把多个文件的内容放到一个文件中？

aws的默认配置

YARN本地日志目录`/var/log/hadoop-yarn/containers`
```
<property>
    <description>Where to store container logs.</description>
    <name>yarn.nodemanager.log-dirs</name>
    <value>/var/log/hadoop-yarn/containers</value>
</property>
```

### 应用调试
在YARN上，Spark Application的Driver和Executors都运行在Container中。在一个Application运行结束后，YARN有两种方式处理container中产生的log：

1. 本地模式
默认情况下，日志会保存在各个机器的本地文件系统上。YARN配置`yarn.nodemanager.log-dirs`指定了container产生的日志的存放位置。有两种查看日志方式：
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

通过增加`yarn.nodemanager.delete.debug-delay-sec`为一个较大的值(如36000)，可以在容器启动的各个节点`yarn.nodemanager.local-dirs`目录保留的应用缓存中查看每个容器的启动环境。这个目录下保存了启动脚本、Jar包、启动时的环境变量等信息。这对调试，特别是Classpath问题很有用。注，这些设置需要集群管理员权限，并且要重启所有的节点管理器，所以在托管集群上很难完成。

有几种可选方式给应用Master和执行器配置定制化的log4j配置：
1. 在使用`spark-submit`提交Spark应用程序时，通过` --files`选项来上传一个配置过的log4j.properties文件
2. 在执行器的`spark.driver.extraJavaOptions`，或在驱动器的`spark.executor.extraJavaOptions` 选项，增加`-Dlog4j.configuration=<file location>`(注，如果使用文件，需要明确指定，并且需要在所有节点存在)
3. 更新`$SPARK_CONF_DIR/log4j.properties`文件，此配置会和其他配置一起自动上传。(注：如果指定了多个选项，另外2个选项的优先级高于本配置)

如果采用第一个方式，需要注意的是，执行器和应用Master使用同一个log4j配置，当他们跑在同一个节点的时候会有问题（如，尝试写同一个日志文件)。

如果需要在YARN上显示、汇聚日志，需要通过在log4j.properties用`spark.yarn.app.container.log.dir`配置相对路径。例如(直接添加下面的):
```
log4j.appender.file_appender.File=${spark.yarn.app.container.log.dir}/tmp/spark.log
```

对于streaming应用，配置`RollingFileAppender`并设置文件路径为YARN的日志目录，可以避免由于日志过大而硬盘空间不够的问题。同时这些log还可以用YARN日志工具访问:

```
log4j.appender.RollingFile.File=${spark.yarn.app.container.log.dir}/tmp/spark.log
```
