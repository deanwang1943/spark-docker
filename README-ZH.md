本文主要是通过docker搭建spark的环境，在docker中运行spark的集群服务。
#1.下载镜像
1.先去docker hub上下载已经集成spark的docker镜像，会方便很多，
地址：https://hub.docker.com/r/singularities/spark/
目前已经更新到spark的2.2的版本，该版本的docker镜像是从hadoop2.8的版本中引申而来。
```shell
docker pull singularities/spark
```
>最新的Docker file可以去github的[地址](https://github.com/SingularitiesCR/spark-docker)查看，下载时间比较长
------
#2.运行脚本
在docker的镜像中包含一个start-spark的shell脚本，这个脚本主要用于初始化mask和workers
##HDFS user
这个运行脚本需要一个HDFS user的用户，该用户会被自动创建
##启动一个master
运行：
```shell
start-spark master
```
如果想通过daemon的方式运行，将daemon作为最后一个参数传入
```shell
start-spark master daemon
```
##启动一个worker
运行脚本启动一个worker
```shell
start-spark worker [MASTER]
```

#3.通过docker compose来编排集群
可以通过docker compose组建来编排一个集群的spark服务，使用配置docker-compose.yml的文件
```yml
version: "2"

services:
  master:
    image: singularities/spark
    command: start-spark master
    hostname: master
    ports:
      - "6066:6066"
      - "7070:7070"
      - "8080:8080"
      - "50070:50070"
    volumes:
      - /home/dean/docker/spark:/opt/hdfs
  worker:
    image: singularities/spark
    command: start-spark worker master
    environment:
      SPARK_WORKER_CORES: 1
      SPARK_WORKER_MEMORY: 2g
    links:
      - master
```
通过命令启动
```shell
docker-compose up [-d]
```
带-d则作为daemon方式启动，编辑hosts文件，加入主机配置
```config
127.0.0.1 localhost
```
Spark Master Web UI：[http://127.0.0.1:8080](http://127.0.0.1:8080)
Spark Worker Web UI：[http://127.0.0.1:50070](http://127.0.0.1:50070)

![Screenshot from 2017-12-17 11-18-00.png](http://upload-images.jianshu.io/upload_images/3111223-13f68ad1d887f488.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

完成集群spark的配置

#4.持久化
镜像中/opt/hdfs为挂载磁盘，master和worker中都存在，为持久话磁盘，对应的主机地址可以使用命令查看docker具体配置信息
```shell
docker inspect [docker id]
```

#5.扩展
可以通过命令的方式增加worker的数量
```shell
docker-compose scale datanode=2
```
#5.Python Demo
```python
from pyspark import SparkContext, SparkConf

url = "spark://master:7077" # url = "spark://172.18.0.2:7077"
conf = SparkConf()
conf.setMaster(url)
conf.setAppName("test application")

sc = SparkContext(conf=conf)
print(sc.version)
```
>url地址为web ui上地址，需要将master的地址陪到hosts，否则无法解析，会无法链接spark，具体的ip在spark的docker启动中不用-d的模式可以看到日志

在系统的hosts文件中
```shell
172.18.0.2       master
```
>使用hdfs文件系统时，报错hdfs.util.HdfsError: Permission denied: user=dr.who, access=WRITE, inode="/test":root:supergroup:drwxr-xr-x  

需要配置hdfs的配置文件，修改可以编辑权限
```config
<property>  
  <name>dfs.permissions</name>  
  <value>false</value>  
</property>  
```
配置文件在/usr/local/hadoop-2.2.0/etc/hadoop/

>如果没有编辑工具，先apt update，在apt install vim

有个错误大意是Http连接太多，没有及时关闭，导致错误 （PS：网上对hdfs操作的资料比较少，大部分都只停留在基础语法层面，但对于错误的记录及解决办法少之又少）
