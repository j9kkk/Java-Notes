[TOC]

# 记一次线上Kafka消费异常排查

## 关键字

`Kafka_2.11_0.10.1.0`|`消费延迟`|`生产积压`|`集群扩容`|`coordinate dead`|`disconnecting from node`|`UnknownHostException`

## 背景说明

### 性能压力

> 20190225因股市行情暴涨引发实际流量激增，带来日志系统整体压力上升，尤其热门服务点日志量超出平时4倍以上，使日志平台数据发生延迟最高达6小时，依此实际情况需要对系统进行瓶颈分析和扩容

----

### 瓶颈定位

> 以目前日志系统架构，分别观察kafka monitor消费偏移量、collector-status、consumer-api日志、elastic-search集群状态及压力发现kafka monitor中消费延迟偏大，es集群压力正常，确认consumer-api日志消费入库存在瓶颈

### 定量分析

> 取20190225 09.11 - 10.40为时间范围统计dc_middleplatform实际处理日志量为28670070条，实际处理能力为112431条/分，此时数据实际延迟166分钟，实际要求处理能力为322135条/分，性能差距约2倍

### 扩容方案

#### 扩容目标

- 扩容后的日志系统可以承受同等的高峰压力，并消除日志数据延迟
- 预留一倍系统容量，用于接入后续新系统

#### 扩容原则

- Kafka的Partition数量最好与Consumer个数保持1:1，否则会造成Partition或Consumer的浪费
- Kafka的每个Partition实际以文件存储在磁盘上，无节制的Partition数量会给Kafka的IO线程带来压力，建议每个Kafka节点的Partition数量不要超过1000
- 对热门服务的压力进行消费能力评估，并依上面的原则汇总Kafka Partition与Consumer配置方案，详见附件【日志系统分区统计】

#### 扩容步骤

Kafka集群扩容

- 从已有的Kafka集群节点上复制程序包及配置文件至新节点

- 修改server.properties中的broker.id为唯一节点标识号

- 修改server.properties中的hostname为新节点的ip

- 执行以下命令启动Kafka新节点

  ```
  bin/kafka-server-start.sh -daemon config/server.properties
  ```

- 启动所有新节点后在KafkaMonitor中查看集群拓扑图是否正常

- 若拓扑图正常，则在任一kafka节点程序目录下逐行执行附件【kafka-alter-partition】中的命令，对现有的分区进行逐一调整

- 调整完毕后进入Kafka Monitor进行检查是否生效

consumer扩容

- 在已有dclogs-consumer-api的服务器上执行如下命令，停止服务

  ```
  ./stop.sh
  ```

- 备份已有的配置文件conf/dclogs-consumer.properties

- 使用同名附件替换conf/dclogs-consumer.properties

- 复制程序包及配置文件至新机器

- 使用如下命令逐台启动服务

  ```
  ./start.sh
  ```

- 观察日志平台及Kafka Monitor是否正常

## 问题描述

按步骤扩容完毕后，发现系统出现如下问题

### 分区丢失

通过`kafka monitor`管理工具查看对应的Topic，发现部分Topic的分区不连续，存在部分丢失的情况

![1552635278419](C:\Users\ftp\AppData\Roaming\Typora\typora-user-images\1552635278419.png)

### 消费延迟

查看部分Topic的Lag延迟，发现扩容后不降反升

### 分区失效

观察Last Seen即最后一次消费时间，发现部分分区的消费Offset在一定时间后停止

### 生产积压

观察collector接口的status，响应结果示例

```
{
	"data": {
		"taskInfo": {
			"activeCount": 17,
			"completedTaskCount": 77111886,
			"corePoolSize": 32,
			"queuedTasks": 1459432,
			"taskCount": 77111904,
			"terminated": false
		}
	},
	"msg": "success",
	"status": 0,
	"time": 0
}
```

发现queuedTasks积压任务一直在增大即生产发生积压

## 故障排查

### Consumer日志

> 根据Topic关键字在消费端查询对应关键流程日志，过滤分析发现如下问题

> 当日志文件过大时，可以使用`tail -n 10000 log.log >out `将最新的10000条日志导出再查看；或在服务器上使用less + reg正则搜索

#### 循环获取Topic分区表信息

Consumer每隔一秒即发起Sending metadata request请求分区表信息，关键日志如下

```
2019-03-15 09:28:33 org.apache.kafka.clients.consumer.internals.AbstractCoordinator [kafka-coordinator-heartbeat-thread | DC_MiddlePlatform]-[DEBUG] Sending coordinator request for group DC_MiddlePlatform to broker 10.207.142.31:9092 (id: 1 rack: null)

2019-03-15 09:28:33 org.apache.kafka.clients.consumer.internals.AbstractCoordinator [pool-1-thread-71]-[DEBUG] Received group coordinator response ClientResponse(receivedTimeMs=1552613313383, disconnected=false, request=ClientRequest(expectResponse=true, callback=org.apache.kafka.clients.consumer.internals.ConsumerNetworkClient$RequestFutureCompletionHandler@fe1332a, request=RequestSend(header={api_key=10,api_version=0,correlation_id=615,client_id=consumer-71}, body={group_id=DC_MiddlePlatform}), createdTimeMs=1552613313283, sendTimeMs=1552613313283), responseBody={error_code=0,coordinator={node_id=3,host=10.207.142.33,port=9092}})

2019-03-15 09:28:33 org.apache.kafka.clients.consumer.internals.AbstractCoordinator [pool-1-thread-71]-[INFO] Discovered coordinator 10.207.142.33:9092 (id: 2147483644 rack: null) for group DC_MiddlePlatform.

2019-03-15 09:28:33 org.apache.kafka.clients.NetworkClient [pool-1-thread-27]-[DEBUG] Sending metadata request {topics=[DC_MiddlePlatform]} to node 3

2019-03-15 09:28:33 org.apache.kafka.clients.Metadata [pool-1-thread-27]-[DEBUG] Updated cluster metadata version 321 to Cluster(id = JrEJbzxNS9yk9cHOHSJT7A, nodes = [10.207.145.13:9092 (id: 6 rack: null), 10.207.145.12:9092 (id: 5 rack: null), 10.207.142.32:9092 (id: 2 rack: null), 10.207.145.11:9092 (id: 4 rack: null), 10.207.142.31:9092 (id: 1 rack: null), 10.207.145.14:9092 (id: 7 rack: null), 10.207.142.33:9092 (id: 3 rack: null)], partitions = [Partition(topic = DC_MiddlePlatform, partition = 80, leader = 4, replicas = [3,4,], isr = [4,]), Partition(topic = DC_MiddlePlatform, partition = 14, leader = 3, replicas = [3,7,], isr = [3,]), Partition(topic = DC_MiddlePlatform, partition = 47, leader = 6, replicas = [5,6,], isr = [6,])...])
```

初步分析得出如下结论：

* 新增的kafka集群节点感知正常，broker.id设置正确
* 消费组协调者coordinator感知正常，Discovered正常
* partitions分区表元信息列表正常，总数与实际情况相符
* partitions分区分散至所有kafka集群节点，且不存在分配异常的leader

结合上面的问题描述，可以分析得知在`kafka monitor`中***丢失的分区***并不是kafka对分区的分配发生了异常，而是对应的分区没有可用的消费者，进而没有对应的`__consumer_offset`记录导致不显示对应分区

#### Kafka节点离线

在获取分区表信息后，需要获取每个分区的偏移量，关键日志如下

```
2019-03-15 09:28:33 org.apache.kafka.common.network.Selector [kafka-coordinator-heartbeat-thread | DC_MiddlePlatform]-[DEBUG] Created socket with SO_RCVBUF = 65536, SO_SNDBUF = 124928, SO_TIMEOUT = 0 to node 4

2019-03-15 09:28:33 org.apache.kafka.clients.NetworkClient [kafka-coordinator-heartbeat-thread | DC_MiddlePlatform]-[DEBUG] Completed connection to node 4

2019-03-15 09:28:34 org.apache.kafka.clients.NetworkClient [pool-1-thread-17]-[DEBUG] Disconnecting from node 4 due to request timeout.

2019-03-15 09:28:34 org.apache.kafka.clients.consumer.internals.ConsumerNetworkClient [pool-1-thread-17]-[DEBUG] Cancelled LIST_OFFSETS request ClientRequest(expectResponse=true, callback=org.apache.kafka.clients.consumer.internals.ConsumerNetworkClient$RequestFutureCompletionHandler@34e88555, request=RequestSend(header={api_key=2,api_version=1,correlation_id=695,client_id=consumer-17}, body={replica_id=-1,topics=[{topic=DC_MiddlePlatform,partitions=[{partition=31,timestamp=-1}]}]}), createdTimeMs=1552613009075, sendTimeMs=1552613009076) with correlation id 695 due to node 4 being disconnected

2019-03-15 09:28:34 org.apache.kafka.clients.consumer.internals.AbstractCoordinator [kafka-coordinator-heartbeat-thread | DC_MiddlePlatform]-[INFO] Marking the coordinator 10.207.142.33:9092 (id: 2147483644 rack: null) dead for group DC_MiddlePlatform
```

由日志得知从node 4获取消费者偏移量时因超时断开链接，导致消费组分配分区失败，进而判断协调者失效并标记为dead；在此状态之后再次发起Sending coordinator request、Sending metadata request等流程并同样堵塞在LIST_OFFSETS阶段，初步得出结论

* 消费节点与node4节点端口可正常建立链接
* 消费节点从node4节点获取消费偏移量时超时导致整个流程失败，并不断重试->失败->重试

### Kafka日志

> 查询node4节点上的${kafka directory}/logs/server.log，重点查看启动时日志发现如下问题

#### UnknownHostException

```
[2019-03-14 16:57:07,700] ERROR Processor got uncaught exception. (kafka.network.Processor)
java.lang.ExceptionInInitializerError
	at kafka.network.RequestChannel$Request.<init>(RequestChannel.scala:114)
	at kafka.network.Processor$$anonfun$processCompletedReceives$1.apply(SocketServer.scala:492)
	at kafka.network.Processor$$anonfun$processCompletedReceives$1.apply(SocketServer.scala:487)
	at scala.collection.Iterator$class.foreach(Iterator.scala:893)
	at scala.collection.AbstractIterator.foreach(Iterator.scala:1336)
	at scala.collection.IterableLike$class.foreach(IterableLike.scala:72)
	at scala.collection.AbstractIterable.foreach(Iterable.scala:54)
	at kafka.network.Processor.processCompletedReceives(SocketServer.scala:487)
	at kafka.network.Processor.run(SocketServer.scala:417)
	at java.lang.Thread.run(Thread.java:745)
Caused by: java.net.UnknownHostException: EM-6CU4525DYP: EM-6CU4525DYP: unknown error
	at java.net.InetAddress.getLocalHost(InetAddress.java:1505)
	at kafka.network.RequestChannel$.<init>(RequestChannel.scala:40)
	at kafka.network.RequestChannel$.<clinit>(RequestChannel.scala)
	... 10 more
Caused by: java.net.UnknownHostException: EM-6CU4525DYP: unknown error
	at java.net.Inet4AddressImpl.lookupAllHostAddr(Native Method)
	at java.net.InetAddress$2.lookupAllHostAddr(InetAddress.java:928)
	at java.net.InetAddress.getAddressesFromNameService(InetAddress.java:1323)
	at java.net.InetAddress.getLocalHost(InetAddress.java:1500)
	... 12 more
```

从中可以发现node 4节点在启动时报出了一个找不到`EM-6CU4525DYP`这个host的异常，在node 4上执行`hostname`命令，发现其主机名即为`EM-6CU4525DYP`!

分析原因，因为新增的四台节点为虚拟机，其hostname默认为对应的虚拟机名称，而kafka启动时会通过`java.net.InetAddress.getLocalHost`方法获取本机IP，当碰到无法解析的hostname时则抛出异常，进而无法提供服务

## 解决方案

* 通过`hostname`命令查看kafka节点的主机名
* 修改/etc/hosts文件，新增对应的映射规则

## 总结

* centos系统的hostname在`/etc/sysconfig/network`文件中

* 线上环境为kafka_2.11_0.10.1.0，验证存在hostname解析异常，在开发环境控制变量验证kafka_2.11_1.1.0不存在hostname解析异常；查看源码发现0.10.1.0使用`java.net.InetAddress.getLocalHost()`获取监听地址，1.1.0使用`new java.net.InetSocketAddress()`

* 使用1.0.0及以下版本时，添加host.name或修改advertisedListener均无效，建议首先在系统HOSTS文件中添加对应的DNS解析

## 资源链接

[Kafka发布包下载](http://kafka.apache.org/downloads)

[Kafka 0.10.1.0 - RequestChannel.scala github](https://github.com/cloudera/kafka/blob/cdh5-base-0.10.0/core/src/main/scala/kafka/network/RequestChannel.scala)

[Kafka 1.1.0 - RequestChannel.scala github](https://github.com/cloudera/kafka/blob/kafka1.0.0-release/core/src/main/scala/kafka/network/RequestChannel.scala)

[如何理解localhost，127.0.0.1，0:0:0:0三者区别](https://stackoverflow.com/questions/20778771/what-is-the-difference-between-0-0-0-0-127-0-0-1-and-localhost)