## Spark基础知识
[大佬论文](http://static.usenix.org/legacy/events/hotcloud10/tech/full_papers/Zaharia.pdf)   
[RDD基本算子大全](http://lxw1234.com/archives/2015/07/363.htm)  

### Spark基础知识
#### 基本概念
>* RDD:弹性分布式数据集。Spark通过使用Spark的转换API，可以将RDD封装为一系列具有血缘关系的RDD，也就是DAG。只有通过Spark的动作(Action)API才会将RDD及其DAG提交到DAGScheduler。
RDD的祖先一定是一个跟数据源相关的RDD，负责从数据源迭代读取数据。
>* DAG: 有向无环图用来反映各RDD之间的依赖或血缘关系。
>* Partition: 数据分区，一个RDD的数据可以划分为多个分区，Spark根据Partition的数量来确定Task的数量。
>* NarrowDependency: 窄依赖，子RDD依赖于父RDD中固定的Partition。分为OnetoOneDependency和RangeDependency。  
>* ShuffleDependency: Shuffle依赖，子RDD对父RDD中所有的Partition都可能产生依赖。具体取决于分区计算器(Partitioner)算法。
>* Job: 用户提交的作业： 当RDD及其DAG被提交给DAGScheduler调度后，DAGScheduler会将所有RDD中的转换及动作视为一个Job。一个Job由一到多个Task组成。
>* Stage: Job的执行阶段。DAGScheduler按照ShuffleDependency作为stage的划分节点对RDD的DAG进行stage划分。
>* Task: 具体执行任务。一个Job在每个Stage内都会按照RDD的Partition数量，创建多个Task。Task分为ShuffleMapTask和ResultTask两种(和stage对应，类似于hadoop中的Map任务和Reduce任务)。
>* Shuffle：Shuffle是所有MapReduce计算框架的核心执行阶段，Shuffle用于打通map任务(ShuffleMapTask)的输出和reduce任务(ResultTask)的输入，map任务的中间输出结果按照指定的分区策略(例如，按照key值哈希)
分配给处理某一分区的reduce任务。 

#### 集群概念
>* 一个RDD分为多个Partition   
>* 每个Partition对应一个Task   
>* 一个文件里有多个Block  
>* 一个InputSplit对应多个Block（意味着一个文件有多个InputSplit）  
>* 一个Task对应一个InputSplit   
>* 具体的Task会被分配到集群上的某个Executor去执行   
>* 每个节点可以起一个或多个Executor   
>* 每个Executor由若干core组成， 每个Executor的每个core一次只能执行一个Task(core是虚拟的core而不是及其的物理CPU核，可以理解为Executor的一个线程)   
>* 每个Task执行结果就是产生了目标RDD的一个Partition   

Task执行的并发度=Executor的数目 * 每个Executor的core数目  
Partition和Task一一对应，而Task和InputSplit一一对应，以sc.textfile为例，文件被划分为多少InputSplit，就有多少Task/Partition。

#### 特点
>* 减少磁盘I/O
>* 增加并行度(理解DAG和Stage)
>* 避免重新计算：当Stage中某个分区的task执行失败后，会重新对此Stage调度，但在重新调度的时候会过滤已经执行成功的分区任务
>* 可选的Shuffle排序： Hadoop的MapReduce在Shuffle之前有着固定的排序操作，而Spark则可以根据不同场景选择在map端排序还是reduce端排序
>* 灵活的内存管理策略： 堆上的存储内存，堆外的存储内存，堆上的执行内存，堆外的执行内存4个部分。提供了执行内存和存储内存之间的"软"边界。Spark的内存管理器提供的Tungsten实现了一种与操作系统
的内存Page非常相似的数据结构，用于直接操作操作系统内存，节省了创建Java对象在堆中占用的内存，使得Spark对内存的使用效率更加接近硬件。Spark会给每个Task分配一个配套的任务内存管理器，对Task粒度
的内存进行管理。

其他特点：
1. [checkpoint机制](https://toutiao.io/posts/x6ugpj/preview)

### RDD编程模型 
RDD(弹性分布式数据集)直接在编程接口层面提供了一种高度受限的共享内存模型，本质是一种分布式的内存抽象，表示一个只读的数据分区集合。一个RDD通常由其他RDD转换而来。RDD定义了各种丰富的转换操作(map,join,filter)
，通过这些转换操作，新的RDD包含了如何从其他RDD衍生所必需的消息，这些信息构成了RDD之间的依赖关系。依赖具体分为两种：窄依赖----RDD之间分区是一一对应的，宽依赖----下游RDD与上游RDD(父RDD)的每个分区都有关，是多对多的关系。
窄依赖中的所有转换操作可以通过pipeline的方式执行，宽依赖需要在不同的节点之间shuffle传输。RDD操作算子：transformation(用来将RDD转换，构建RDD的构建关系)和action(用来触发RDD的计算，包括show,count,collect,saveAsTextFile)。
**总结：**  RDD的计算任务可以被描述为：从稳定的物理存储中加载记录，记录被传入由一组确定性操作构成的DAG，然后写回稳定存储。RDD还可以将数据集缓存到内存中，使得多个操作之间能够很方便地重用数据集。   
**容错性方面**，RDD通过Lineage信息(血缘关系)来完成容错，即使出现数据分区丢失，也可以通过Lineage信息重建分区。如果在应用程序中多次使用同一个RDD，则可以将这个RDD缓存起来，该RDD只有在第一次计算通过Lineage信息得到分区数据，
在后续其他地方用到这个RDD时就会从缓存中读取。Lineage信息虽然实现了容错，但是对于长时间的迭代应用来说，对着迭代的进行，RDD之间的Lineage信息会越来越长，一旦后续迭代出错，需要非常长的Lineage信息去重建，对性能影响很大。
为此，RDD支持用checkpoint机制将数据保存到持久化的存储中，checkpoint后的RDD不需要知道它的父RDD，可以从checkpoint出获取信息。
```text
def main(args: Array[String]) {
   val conf = new SparkConf()
   val sc = new SparkContext(conf)
   val result = sc.textFile("hdfs://...").flatMap(line => line.split(" ")).map(word => (word, 1)).reduceByKey(_ + _)
   result.collect()  //直到collect，整个DAG才开始调度执行
}
``` 
在RDD基础上，提供了DataFrame和DataSet用户编程接口。DF和RDD一样，都是不可变分布式弹性数据集。不同之处：RDD中的数据不包含任何结构信息，需要开发人员使用特定函数完成数据结构的解析；而DataFrame中的数据集按列名存储，
具有Schema信息。Dataset是DF的扩展，DF本质上是一个特殊的Dataset。Dataset引入了编码器(Encoder)的概念。Encoder不仅能够在编译阶段完成类型安全检查，还能生成字节码与堆外数据进行交互，提供对各个属性的按需访问，而不必对
整个对象进行反序列化操作，极大减少了网络传输的代价。
```text
case class Person(name: String, age: Long)
val caseClassDS = Seq(Person("Andy", 32)).toDS()
caseClassDS.show()
```
