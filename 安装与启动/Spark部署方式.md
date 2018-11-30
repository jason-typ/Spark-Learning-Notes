---
title: Spark部署方式
categories: Spark
tags: Spark Data
---

<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->
<!-- code_chunk_output -->

* [local模式](#local模式)
* [Standalone模式](#standalone模式)
		* [1 standalone模式启动集群](#1-standalone模式启动集群)
		* [2 连接到cluster](#2-连接到cluster)
		* [3 环境变量配置](#3-环境变量配置)
		* [4 启动spark程序](#4-启动spark程序)
* [Hadoop YARN模式](#hadoop-yarn模式)

<!-- /code_chunk_output -->


Spark可以部署在一台计算机上，即local模式，也可以部署在多台计算机上，即cluster模式。在集群上运行时，根据提供资源调度的组件的不同，又可以细分为Standalone模式、Hadoop YARN模式、Apache Mesos模式和Kubernetes模式。

### local模式

local模式就是部署在一台机计算机上的模式，通常用于非生产环境的测试。local模式下还可以根据计算运行在几个线程中来进一步细分：

- `local`：所有计算运行在一个线程中，没有任何并行计算
- `local[n]`: 指定用几个线程来运行计算，比如local\[4]表示运行4个worker线程。通常根据我们cpu核数来指定，有多少个核就指定几线程。
- `local[*]`: 直接按照CPU的核数来设置线程数

在IDEA中使用`local`模式，直接执行Spark程序如：

```
object spark_learning {
  val logger = Logger(this.getClass)
  def main(args: Array[String]): Unit = {
    val conf = new SparkConf().setAppName("spark-learning").setMaster("local")
    val sc = new SparkContext(conf)
    val lines = sc.textFile("/usr/local/spark/README.md")
    logger.info(lines.count().toString)
  }
}
```

这里遇到一个小坑，目前Scala的最新版本已经到了2.12，但是Spark编写所使用的Scala版本仍然停留在2.11。如果sbt中指定的Scala的版本过高，会导致执行失败。只需要降低编译所使用的Scala版本，sbt文件如下：

```
val spark_core = "org.apache.spark" % "spark-core_2.11" % "2.4.0"
val logback = "ch.qos.logback" % "logback-classic" % "1.2.3"
val scala_logging = "com.typesafe.scala-logging" %% "scala-logging" % "3.9.0"

lazy val spark_learning = (project in file(".")).settings(
  name := "spark-learning",
  scalaVersion := "2.11.12",
  version := "0.1",
  libraryDependencies ++= Seq(logback, scala_logging, spark_core)
)
```

### Standalone模式

Spark本身提供了一个资源管理调度服务，叫做`Standalone`。此种模式下，有一个`Master`节点和若干个`Slave`节点。可以使用Spark提供的脚本完成各种节点的启动工作。

- `sbin/start-master.sh`: 在脚本执行的机器上启动一个master实例
- `sbin/start-slaves.sh`: 在`conf/slaves`中指定的机器上，分别启动一个Worker实例。因此需要创建一个`conf/slaves`文件，并在其中加入想要启动Worker实例的机器的地址，每个地址一行。如果没有这个文件，默认会在本地启动一个worker实例
- `sbin/start-slave.sh`: 在本地启动一个worker实例。实际上上面的`start-slaves.sh`文件最终就是调用的这个文件完成真正的启动的。
- `stop-xxx.sh`: 停止对应的master，或slave，或全部停止

##### 1 standalone模式启动集群

- 首先在想要运行master的机器上执行`start-master.sh`脚本。执行结束后，sparkMaster会默认在本机的7077端口上提供服务。另外，spark提供了webUI服务，供网页端查看运行状态，端口默认是8080。
- 然后在conf目录下创建slaves文件，文件中加入想要启动worker实例的节点的地址，如`Jason`
- 运行脚本`start-slaves.sh`。这个脚本会根据`conf/slaves`文件中指定的地址，每个地址上启动一个worker实例。脚本需要传入master节点的地址和服务端口。如：`./start-slaves.sh jason:7077`
- 在master提供的webUI上查看，是否成功注册了一个worker。并且可以看到worker节点的信息，如CPU核数，内存大小

上面也可以直接使用`start-all.sh`这个脚本，只需要对`slaves`文件配置好就可以。以上步骤会发现，使用standalone跟再Hadoop 1.0中的使用方式很相似，基本上是一样的。

另外，执行这些脚本时要注意当前所处的机器，是不是确实是在对的那台机器上运行的。还有，在启动的过程中，网络的设置很重要，不然很容易就会出现莫名其妙的问题。我的本地的localhost叫做`jason`，因此上面在部署的时候都是统一使用的这个(可以使用命令`hostname`查看本机的hostname)。

##### 2 连接到cluster

  在cluster上运行应用程序，需要传入master的地址和端口，形式为`spark://IP:port`。比如在cluster上运行一个spark shell:

```
./spark-shell --master spark://jason:7077
```

此时在master的webUI上查看，一个正在running的appication的名字为Spark Shell，说明目前的Spark Shell确实是运行在集群上的。

##### 3 环境变量配置

Spark同样严重依赖于环境变量的配置。以上已经可以正常的启动一个使用standalone的cluster了，如果想要进一步配置，可以修改`conf/spark-env.sh`文件。(spark默认提供了一个副本，拷贝一下即可)。下面是其中的一些配置的含义

- SPARK_MASTER_HOST 将master节点绑定到一个指定的IP地址上
- SPARK_MASTER_PORT 在指定的端口上提供Master服务(默认是7077)
- SPARK_MASTER_WEBUI_PORT 在指定端口上提供Master webUI服务(默认是8080)
- 待补充

##### 4 启动spark程序

使用`spark-submit`脚本是提交应用程序到cluster最直接的方式。具体参考《提交appication.md》。

### Hadoop YARN模式
