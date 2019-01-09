## RDD编程模型 
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
