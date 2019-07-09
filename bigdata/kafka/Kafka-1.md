本文是Apache Kafka实战的学习笔记
## 认识Kafka
### 操作
>* 启动zk
>* 启动kafka(在每个节点执行)kafka-server-start.sh  -daemon config/server.properties &
>* 创建topic：kafka-topic.sh --create --topic jinx01 --partitions 1 --replication-factor 2 --zookeeper hd1:2181,hd2:2181(这个写一个运行状态的zk节点即可)
>* 查看所有的topic: kafka-topic.sh --list --zookeeper hd1:2181,hd2:2181
>* 查看某一topic状态：kafka-topic.sh --describe --zookeeper hd1:2181,hd2:2181 --topic jinx01
>* 发送消息：kafka-console-producer.sh --broker-list hd1:9092 --topic jinx01  
   haha  
   hehe  
   eeee  
>* 消费消息：kafka-console.consumer.sh --topic jinx01 --zookeeper hd1:2181

### 消息引擎系统
>* 消息设计：SOAP协议采用XML，Web Service支持Json，Kakfa是二进制方式保存的。
>* 传输协议设计：自己设计了一套二进制消息传输协议
>* 消息引擎范型： 消息队列模型和发布订阅模型，kafka通过consumer group同时支持这两种模型。

### **基本概念和术语**
kafka是一个消息引擎系统，也是一个流数据处理平台。  
>* 消息：由消息头、key、value组成。Kafka使用ByteBuffer(二进制字节数组)而不是独立对象，从而避免了堆上分配内存，节省了内存空间，同时页缓存的好处还有：broker崩溃时，
堆内存数据消失，但是页缓存还在。
>* topic和partition：kafka采用topic-partition-message的三级结构来分散负载。partition内的每条消息都有一个递增的序列号(offset)。消费者端也有offset概念，指的是消费的进度。
>* replica： 就是分区的副本，包括leader replica和follower replica，后者不提供服务只是被动地向前者获取数据，当前者所在broker宕机，就会从剩余的replica中选取新的leader。
kafka保证一个partition的多个replica不会分配在同一个broker上。
>* ISR: 与leader replica保持同步的replica集合，进度滞后到一定程度的replica将会被踢出ISR。只有该集合内所有replica都接受到了同一条消息，Kafka才会将该消息置于已提交状态(发送成功)。

### **kafka概要设计**
####吞吐量/延时
如果采用批处理思想，吞吐量大增但是延时就高，需要权衡。  
Kafka如何做到高吞吐量和低延时？  
Kafka写入(通过追加方式)很快，因为并没有直接将数据持久化到磁盘，而是先把数据写入到操作系统的页缓存中，然后由操作系统决定何时写回磁盘。   、
读取首先尝试从OS的页缓存上读取，命中遍把消息经页缓存直接发送到网络socket上。使用Linux的sendfile系统调用，基于[零拷贝技术](https://www.jianshu.com/p/fad3339e3448)。  
总结，Kafka如何实现高吞吐量和低延迟：
>* 大量使用页缓存
>* 操作系统完成物理I/O
>* 追加写入方式
>* sendfile零拷贝技术  

#### 消息持久化
消息持久化到磁盘的好处有两点：
>* 解耦了生产者和消费者
>* 可以方便kafka下游系统对消息进行处理

#### 负载均衡和故障转移
通过领导者选举实现负载均衡，均匀分散各个partition的leader；   
会话机制通过zk来检测节点失效。  

#### 伸缩性  
线性扩容的阻碍之一就是状态的保存，通过zk来保存管理系统的状态。

