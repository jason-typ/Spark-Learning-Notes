SparkSQL是一个用于处理结构化数据的Spark组件。SparkSQL强调的是结构化数据，而非SQL。

与Spark的RDD API不同，Spark SQL提供的接口包含了更多数据的结构信息，以及将要在数据上执行的计算。在内部，Spark SQL会利用这些额外的信息，来做一些优化的工作。

SparkSQL提供了结构化的API(Structured API，基于RDD的底层API称为非结构化API，UNStructured API)，操作的数据结构是DataFrame/DataSet(也可以直接执行SQL语句，得到`DataFrame`类型数据)。但无论怎样(API不同或是编程语言不同)，如下所述，最终都会通过Analyzer变为在RDD上执行的操作。




SQL语句经过解析器之后，会被解析成一棵树状结构。这棵树包含很多节点对象，每个节点都拥有特定的数据类型，同时会有0个或多个孩子节点。节点对象在代码中定义为`TreeNode`。

![SparkSQL执行过程](/images/2019/03/sparksql执行过程.png)

1. SqlParser对SQL语句进行词法和语法解析，生成UnresolvedLogicalPlan
  这一步完成对SQL语句的解析，确保了语法上没有问题，不错此时语句中引用的属性一般都是Unresolved。一个属性，在不知道它的类型，或没有把它关联到一个表(别名)上时，称它为Unresolved的。
  比如一条简单的查询语句，`SELECT col FROM sales`，`col`属性的类型，甚至是`col`这个属性是否存在，`sales`这个表是否存在，目前都还是不确定的。
2. Analyzer使用Analysis Rules，利用元数据(Catalog)，对属性完成解析，得到Logical Plan。
  Spark中使用Catalog管理数据的元数据
3. Optimizer使用Optimization Rules将LogicalPlan进行优化，得到Optimized Logical Plan
4. Planner使用Planning Strategies，根据Optimized Logical Plan转换为Physical Plan。
  SQL是结果描述性语言，因此对于相同的结果，可以有不同的执行过程，所以可能会生成多个Physical Plan供选择。根据一个CostModel从中选取一个最优的，使用这个Selected Physical Plan执行RDD的计算。


## Structured API的执行

执行大致可以分为三步：
1. 将用户编写的基于DataFrame/DataSet/SQL的代码转换为Logical Plan
2. 将Logical Plan转换为Physical Plan(基于RDD的操作)
3. 在Cluster上执行Physical Plan

### 1. Logical Planning

分为三个步骤：
1. 首先分析代码，检查代码的合法性，将代码变为unresolved logicical plan
  unresolved，表达的意思是不知道属性的类型，或者还没有确认某个table中确实有这个属性。如

2. 检验用户代码中用到的数据是否合法(比如可能用到的表并不存在)。DataSet/DataFrame支持结构化数据，很重要的方面就是元数据的管理。Spark中所有的原数据都通过Catalog来管理。因此这一步中，Analyzer使用Catalog中的元数据验证数据源是否合法
3. 将resolved logical plan传给Catalyst，对执行步骤进行优化(比如一些pushDown操作)，得到Optimized Logical Plan

### 2. Physical Planning
得到Optimized Logical Plan后，进一步得到Physical Plan(又称为Spark Plan)。分为三个步骤：
1. 根据Optimized Logical Plan得到Physical Plans。对于同样的结果，会有不同的执行步骤，因此这里会生成多个Physical Plans供选择
2. 通过一个Cost Model选出上面Physical Plans中开销最小的一个plan
3. Physical Plan在cluster上执行(对RDD的操作)

### 3. 执行

Physical Plan对应在RDD上的一系列操作。
