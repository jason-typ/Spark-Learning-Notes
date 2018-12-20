Spark SQL是Spark用来处理结构化数据的模块。与Spark的RDD API不同，Spark SQL提供的接口包含了更多数据的结构信息，以及将要在数据上执行的计算。在内部，Spark SQL会利用这些额外的信息，来做一些优化的工作。

有多重方式使用Spark SQL，包括SQL、DataSet/DataFrame API。但无论使用哪种API或是编程语言，最终都是基于同样的执行引擎。
