[api查看](https://spark.apache.org/docs/latest/streaming-programming-guide.html#input-dstreams-and-receivers)

#### 流式状态管理操作
具体查看《流式状态管理.md》

#### 窗口操作
SparkStreaming提供了对一个滑动窗口中的数据进行操作的支持，如下图所示。
![SparkStreaming滑动窗口](/images/2019/02/sparkstreaming滑动窗口.png)

窗口沿着DStream滑动，每次落在窗口内的数据会组合在一起，执行定义的操作。窗口操作需要两个时间参数：
- **window length**，即窗口大小，多长时间内的数据作为一个窗口
- **sliding interval**，滑动间隔，多长时间后窗口向下一个地方滑动

这两个时间参数必须是batch interval的整数倍。图中的窗口大小为3个batch interval，滑动间隔为2个batch interval。

还是以wordcount为例，假设想要每10s，统计一下过去30s内的单词出现次数。可以使用`reduceByKeyAndWindow`函数：
```
// Reduce last 30 seconds of data, every 10 seconds
val windowedWordCounts = pairs.reduceByKeyAndWindow((a:Int,b:Int) => (a + b), Seconds(30), Seconds(10))
```
