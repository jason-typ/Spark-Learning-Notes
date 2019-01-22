## Spark历史服务器
Spark Application在运行期间会提供一个WebUI来查看具体的执行细节。但是在Application运行结束后，该Web页面就失效了，无法继续查看该Application的历史记录。因此才需要一个历史服务器。

### 启动历史服务器
Spark提供`./start-history-server.sh`脚本用于启动历史服务器，但需要先配置好历史服务器读取日志的目录(这个目录必须事先存在)：

```
spark.history.fs.logDirectory=file:///tmp/eventLog
```
历史服务器的默认端口是18080，启动后可以在这个端口上看到历史服务器。
### 历史服务器的相关配置

- `spark.eventLog.enabled`，是否记录Spark events，打开后才会记录Spark Application的events，这样在程序运行结束后才能利用历史服务器查看该APP的运行信息
- `spark.eventLog.dir`，Spark events存放的地址
- `spark.eventLog.compress`，是否压缩events
- `spark.eventLog.overwrite`，是否覆写已经存在的文件
- `spark.eventLog.buffer.kb`，写入到输出流时，用到的buffer的大小
- 其他 [History Server Configuration](https://spark.apache.org/docs/latest/monitoring.html)

**`spark.eventLog.dir`与`spark.history.fs.logDirectory`的区别**
`spark.history.fs.logDirectory`指定了历史服务器从何处读取数据，`spark.eventLog.dir`指定了Application运行中产生的events存放在哪儿。因此这两者一般需要设置成同样的值。

> spark.eventLog.dir is the base directory in which Spark events are logged, if spark.eventLog.enabled is true. Within this base directory, Spark creates a sub-directory for each application, and logs the events specific to the application in this directory. Users may want to set this to a unified location like an HDFS directory so history files can be read by the history server.
> spark.history.fs.logDirectory is for the filesystem history provider, the URL to the directory containing application event logs to load. This can be a local file:// path, an HDFS path hdfs://namenode/shared/spark-logs or that of an alternative filesystem supported by the Hadoop APIs.
