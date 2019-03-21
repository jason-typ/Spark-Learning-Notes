SQL/DataFrame/DataSet执行过程

SQL经过解析，最终得到Physical Plan，即对RDD的操作。

这一过程中，又可以分为几个步骤，首先是将用户的代码转换为Logical Plan
![Logical_Plan_Process](images/2019/03/logical-plan-process.png)

某个属性是unresolved的，表达的意思是不知道这个属性的类型，或者还没有确认某个table中确实有这个属性。

SparkSQL中有一个Catalog对象
