# Spark启动

Spark需要先启动Master节点，然后Slave节点在启动时需要指定Master节点的地址和端口号，然后注册到Master节点上。

### 单独启动

1. 首先在Master机器上，启动Master节点

  `start-master.sh`
  启动之后，就可以在`http://master-ip:8080`查看Master提供的webUI服务，此时由于没有worker加入进来，因此看到Worker中为空
2. 在Slave机器上，启动Slave节点

  这里启动的时候需要告诉Slave节点，Master节点提供服务所在的地址和端口
  `start-slave.sh spark://master-ip:7077`
  运行命令后查看log，会提示Worker成功注册到Master上。查看Master的webUI服务，可以看到新注册进来了Worker。

### 批量启动

使用start-all.sh脚本。这个脚本其实也是单独调用的start-master.sh和start-slave.sh。
