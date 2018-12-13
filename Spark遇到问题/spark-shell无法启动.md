
### spark-shell无法启动
【问题描述】
下载的Spark源码，使用mvn命令完成编译，并且编译成功。启动spark-shell时报错：
```
Can't assign requested address: Service 'sparkDriver' failed after 16 retries (on a random free port)
```
【修改方法】
系本地的网络配置问题。使用`localhost`命令查看本机的hostname是`Jason`，但是在`/etc/hosts`文件中没有对应的地址。有两个修改方法：
1. 修改/etc/hosts文件，添加hostname对应的地址，如：
    ```
    127.0.0.1   Jason
    ```
2. 修改本机的hostname：
    ```
    sudo hostname -s 127.0.0.1
    ```
3. 如果不想动hostname，可以通过配置环境变量：修改conf文件夹下的spark-env.sh，设置这个节点上Spark所绑定的IP地址。添加配置：
    ```
    export SPARK_LOCAL_IP=127.0.0.1
    ```
【修改结果】
spark-shell正常启动
【问题参考地址】
[Stack Overflow](https://stackoverflow.com/questions/34601554/mac-spark-shell-error-initializing-sparkcontext)

---
