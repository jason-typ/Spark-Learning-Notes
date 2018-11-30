---
title: Spark基本结构
categories: Spark
tags: Spark Data
---


<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->
<!-- code_chunk_output -->

* [Spark基本结构](#spark基本结构)
	* [Spark应用、作业、阶段、任务](#spark应用-作业-阶段-任务)

<!-- /code_chunk_output -->

## Spark基本结构
Spark集群由以下部分组成：

- Dirver Program: 客户端驱动程序，用于将任务程序转换为RDD和DAG，并与cluster master进行通信与调度
- Cluster Master：集群的资源管理器，负责集群资源的分配与管理。Cluster Master是一个独立的部分，目前Spark支持的资源管理器有Standalone、Hadoop YARN、Apache Mesos和Kubernetes。
- Worker：Spark的工作节点
- Executor：运行在Worker上的工作线程，负责具体的计算

![Spark应用程序整体结构](https://spark.apache.org/docs/latest/img/cluster-overview.png)

执行一个程序的基本流程为：

1. 在Driver Program中完成初始化工作(SparkContext)，对任务进行分解，得到一系列的task
2. Driver Program向Cluster Master申请资源(Executor)，Cluster Manager完成Worker Node上的资源的分配
3. Driver Program向Executor分配task，Executor在线程中执行task

Spark集群中同时执行的任务可能会有多个，也就是多个Driver Program，每个Driver Program都有自己**独立**的进程(群，Executors)，每个Executor中又会以多线程的形式完成Task
的执行。

### Spark应用、作业、阶段、任务

**Application**，用户编写的Spark应用程序。分解下来，包括运行在client中的Driver Program和运行在Executors中的代码(driver和executor)。

在Driver(就是我们的main函数)中，需要构建一个Spark运行环境(SparkContext)来完成初始化，进行资源的申请、任务的分配和监控等()。我们知道，Spark应用程序其实就是对RDD的处理。当对RDD执行了一个动作，我们需要一个结果时，就会触发一个**作业**(Job)的提交。因此一个Application中会包含一个或多个Job。

为执行Job，SparkContext会根据RDD之间的依赖关系构建DAG图。之后，SparkContext会将DAG图分解为不同的**stage**(依据依赖的类型)。一个stage中包含对RDD的一次或多次处理，会被分解为一个或多个**task**(分解的依据暂不清楚)，所以stage又被称为**taskSet**。

到这里，已经将用户提交的Application分解为多个可并行执行的任务，SparkContext的初始化工作也就结束了。之后就是SparkContext与ClusterManager的交互通信，申请资源，分配任务的过程。
