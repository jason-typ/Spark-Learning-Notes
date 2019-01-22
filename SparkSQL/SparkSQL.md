Spark SQL是Spark用来处理结构化数据的模块。与Spark的RDD API不同，Spark SQL提供的接口包含了更多数据的结构信息，以及将要在数据上执行的计算。在内部，Spark SQL会利用这些额外的信息，来做一些优化的工作。

SparkSQL提供了结构化的API(Structured API，基于RDD的底层API称为非结构化API，UNStructured API)，操作的数据结构是DataFrame/DataSet(也可以直接执行SQL语句，得到`DataFrame`类型数据)。但无论怎样(API不同或是编程语言不同)，如下所述，最终都会通过Analyzer变为在RDD上执行的操作。

## Structured API的执行

执行大致可以分为三步：
1. 将用户编写的基于DataFrame/DataSet/SQL的代码转换为Logical Plan
2. 将Logical Plan转换为Physical Plan(基于RDD的操作)
3. 在Cluster上执行Physical Plan

### 1. Logical Planning

分为三个步骤：
1. 首先分析代码，检查代码的合法性，将代码变为unresolved logicical plan
2. 检验用户代码中用到的数据是否合法(比如可能用到的表并不存在)。DataSet/DataFrame支持结构化数据，很重要的方面就是元数据的管理。Spark中所有的原数据都通过Catalog来管理。因此这一步中，Analyzer使用Catalog中的元数据验证数据源是否合法
3. 将resolved logical plan传给Catalyst，对执行步骤进行优化(比如一些pushDown操作)，得到Optimized Logical Plan

### 2. Physical Planning
得到Optimized Logical Plan后，进一步得到Physical Plan(又称为Spark Plan)。分为三个步骤：
1. 根据Optimized Logical Plan得到Physical Plans。对于同样的结果，会有不同的执行步骤，因此这里会生成多个Physical Plans供选择
2. 通过一个Cost Model选出上面Physical Plans中开销最小的一个plan
3. Physical Plan在cluster上执行(对RDD的操作)

### 3. 执行

Physical Plan对应在RDD上的一系列操作。
