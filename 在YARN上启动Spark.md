#在YARN上启动Spark

在YARN上启动Spark Job，首先要确保环境变量`HADOOP_CONF_DIR`或`YARN_CONF_DIR`已配置，他们指定Hadoop和YARN配置文件的目录。修改Spark下的`spark-env.sh`文件，添加：

```
export HADOOP_HOME=/usr/local/hadoop
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
export YARN_CONF_DIR=$HADOOP_HOME/etc/hadoop
```

在YARN上部署Spark应用程序有两种模式：YARN-Cluster和YARN-Client。他们的主要区别在于Spark Driver的运行位置不同，或者说`SparkContext`完成初始化的位置不同。在YARN-Cluster模式中，Spark Driver运行在一个ApplicationMaster中，由YARN管理，客户端在提交运行程序所需要的一切之后就可以退出了。在YARN-Client模式中，Spark Driver运行在一个单独的进程中，此时的ApplicationMaster只是用来资源的申请。两种模式的具体区别，参考文章 [Spark On Yarn的两种模式yarn-cluster和yarn-client深度剖析](https://www.cnblogs.com/ITtangtang/p/7967386.html)

在YARN Cluster上提交一个Spark应用程序同样使用`spark-submit`脚本，不同的是，在使用`--master`选项时，只是指定使用`yarn`，而不像其他Cluster一样，在命令中指定一个位置，而是通过Hadoop Configuration来配置的，这也就是在一开始需要配置`HADOOP_CONF_DIR`和或`YARN_CONF_DIR`这两个环境变量的原因。

比如在YARN上以client模式启动spark-shell：

```
spark-shell --master yarn --deploy-mode client
```

比如在YARN上以cluster模式提交一个任务：

```
/usr/local/spark/bin/spark-submit --class com.data.SparkLearning --master yarn --deploy-mode cluster  /Users/tang/project/testspace/spark-learning/target/scala-2.11/spark-learning_2.11-0.1.jar
```

Spark on YARN读取的是HDFS上的文件？

### 添加其他Jar包

在cluster模式下，由于Driver在另一个不同的机器(ApplicationMaster)上执行，因此代码中对SparkContext的初始化中`addJar`操作可能会找不到对应的Jar包。此时需要在启动时通过`--jars`选项主动指定Jar包在本地的位置：

```
./bin/spark-submit --class my.main.Class \
    --master yarn \
    --deploy-mode cluster \
    --jars my-other-jar.jar,my-other-other-jar.jar \
    my-main-jar.jar \
    app_arg1 app_arg2
```

### Spark运行时Jar包准备

要想在YARN上运行Spark程序，首先Spark的版本是需要支持YARN的。另外，为了在YARN中能够访问到Spark的运行时jar包，需要通过spark.yarn.archive或spark.yarn.jars来指定jar包在HDFS中的位置。如果不指定的话，Spark会打包本地under $SPARK_HOME/jars的所有文件，并分发这些jar包。如果已经放在了HDFS上，就不需要每次运行一个程序都分发一次了

### Configuration

Configuration一般都是类似的，
