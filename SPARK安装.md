---
title: Spark安装
categories: Spark
tags: Spark, Data
---


<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->


### Spark安装

1. 下载编译好的spark安装包，并解压到相应目录

<pre>
tar xvz -f spark-2.3.1-bin-hadoop2.7.tgz
sudo mv spark-2.3.1-bin-hadoop2.7.tgz /usr/local
sudo ln -s spark-2.3.1-bin-hadoop2.7.tgz spark
</pre>

2. 配置环境变量

修改`/etc/profile`文件，将spark的路径添加到环境变量中

<pre>
# SPARK VARIABLE START
export SPARK_HOME=/usr/local/spark
export PATH=$PATH:${SPARK_HOME/bin}
export PATH=$PATH:${SPARK_HOME/sbin}
# SPARK VARIABLE END
</pre>

3. 配置spark环境

需要修改两个文件`spark-env.sh`和`slaves`。刚下载下来的spark安装包中有他们的模板文件:slaves.template和spark-env.sh.template。可以将它们重命名（或复制一份）为`spark-env.sh`和`slaves`。

修改`spark-env.sh`，在末尾增加以下内容

### spark本地运行测试

### Spark部署

见《Spark部署方式.md》

### Spark Streaming示例

示例来自Spark Streaming的官方文档，功能是tcp监听本地的某个端口，统计收到的数据的词频。代码如下：

<pre>
import org.apache.spark.storage.StorageLevel
import org.apache.spark.streaming.{Seconds, StreamingContext}
import org.apache.spark.{SparkConf, SparkContext}

// Create a local StreamingContext with two working thread and batch interval of 1 second.
// The master requires 2 cores to prevent a starvation scenario.

val conf = new SparkConf().setMaster("local[2]").setAppName("NetworkWordCount")
val ssc = new StreamingContext(conf, Seconds(1))

val lines = ssc.socketTextStream("localhost", 9999)
val words = lines.flatMap(_.split(" "))
val pairs = words.map(word => (word, 1))
val wordCounts = pairs.reduceByKey(_ + _)
wordCounts.print()
ssc.start()             // Start the computation
ssc.awaitTermination()  // Wait for the computation to terminate
</pre>

首先要创建一个`StreamingContext`对象，它是所有操作的入口。设置conf时，至少要设置两个线程，因为接收网络包就需要一个线程，设置为两个线程来防止`starvation scenario`。利用这个spark streaming context，我们可以创建一个用于网络收包的DStream对象。DStream对象接收包，每行作为一个记录。对每行数据，做一个单词统计。这个时候只要往本机的对应端口发包，就能统计到发送的数据的词频。如

<pre>
nc -lk 9999
hello world
</pre>

### 运行

可以直接在IDE中运行，也可以打成jar包在本地提交运行，命令为`spark-submit --class class_name path_to_jar`
