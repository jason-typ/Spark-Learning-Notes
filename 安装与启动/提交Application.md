## Application的提交

可以使用`bin/spark-submit`脚本在cluster上启动一个程序。这个脚本适用于目前Spark支持的所有的cluster manager(4种)。

1. 打包应用程序

  jar包中包括自己的代码，也需要添加依赖于其他项目的代码(jar包)。sbt和Maven都提供了装配(打包)的插件。对于项目中依赖于Hadoop/Spark的代码，可以标记为`provided`以排除打包，它们会在运行时由cluster manager提供。

2. 提交

  打包好之后，就可以使用`spark-submit`脚本提交jar包。具体的命令为：

  ```
  spark-submit \
  --class <main-class> \
  --master <master-url> \
  --deploy-mode <deploy-mode> \
  --conf <key>=<value> \
  ... # other options
  <application-jar> \
  [application-arguments]
  ```

  测试代码如下：

  ```
  package com.data

  import org.apache.spark.{SparkConf, SparkContext}

  object SparkLearning {
    def main(args: Array[String]) {
      val inputFile =  "/usr/local/spark/README.md"
      val conf = new SparkConf().setAppName("WordCount")
      val sc = new SparkContext(conf)
      val textFile = sc.textFile(inputFile)
      val wordCount = textFile.flatMap(line => line.split(" ")).map(word => (word, 1)).reduceByKey((a, b) => a + b)
      wordCount.foreach(println)
    }
  }
  ```

  使用`sbt package`命令打包jar包，然后使用spark-submit脚本提交jar包：
  ```
  /usr/local/spark/bin/spark-submit --class com.data.SparkLearning --master spark://jason:7077 /Users/tang/project/testspace/spark-learning/target/scala-2.11/spark-learning_2.11-0.1.jar
  ```

  其中`--class`指定main函数，应用程序的入口，`--master`指定master提供服务的地址

### Master URLS
传给Spark的Master URL可以是以下几种形式

- local，在本地，并只以一个工作线程来完成任务
- local\[K]，在本地，以K个工作线程来完成任务
- local\[K, F]，在本地以K个工作线程来完成任务，最多可以失败F次
- local\[*]，在本地，线程数根据主机的核数来确定
- local\[*, F]
- spark://HOST:PORT，连接到指定的Standalone集群来提交工作，端口默认是7077
- spark://HOST1:PORT1, HOST2:PORT2，Standalone集群是主从式，存在单点故障，可以使用Zookeeper搭建一个standby的master，HOST2和PORT2就是standby master的主机地址和端口
- mesos://HOST:PORT，连接到指定的Mesos cluster。端口默认是5050
- YARN 连接到YARN Cluster。模式有两种：client模式和cluster模式。cluster的地址根据环境变量`HADOOP_CONF_DIR`或`YARN_CONF_DIR`来找到
- k8s://HOST:PORT，连接到Kubernets Cluster
