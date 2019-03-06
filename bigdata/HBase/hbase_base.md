基于hbase构建一个对气象数据在线查询的程序  
link: http://www.ncdc.noaa.gov/
HBase是HDFS上面向列的分布式数据库，自底向上构建，可以通过简单增加节点实现线性扩展。不是关系型数据库，不支持SQL。   
应用把数据放在带标签的表中，表格中的单元格有版本号(时间戳).行的键是字节数组，表中的行根据行的键值进行排序。行中的列被分为"列族": 
info:format和info:geo都是列族info的成员，冒号间隔。一个表的列族必须作为表模式定义的一部分预先给出，但新的列族成员可以随后按需加入。     
**准确地说：HBase是一个面向列族的存储器**。调优和存储都是在这个层次上进行的。所以列族的所有成员要有相同的访问模式和大小特征。     
相比RDBMS中的表，HBase的单元格有版本，行是排序的，只要列族预先存在，客户端随时可以把列添加到列族中。  
概念： 区域是HBase集群上分布数据的最小单位，由表中行的子集构成。
**Region虽然是分布式存储的最小单元，但并不是存储的最小单元。Region由一个或者多个Store组成，每个store保存一个columns family；
每个Strore又由一个memStore和0至多个StoreFile组成，StoreFile包含HFile；memStore存储在内存中，StoreFile存储在HDFS上。**
无论对行进行访问的事务牵涉多少列，对行的更新都是原子的。  
master+regionserver  master协调管理regionserver，负载很轻，regionserver负责0/多个区域的管理以及响应客户端的读写请求以及负责区域划分。 
zookeeper负责管理诸如hbase:meta目录表的位置以及当前集群主控机地址等消息，如果区域分配过程中有服务器崩溃，可以通过zookeeper来进行分配的协调。
配置文件conf下面。   
新连接到zookeeper集群上的客户端首先查找hbase:meta位置，然后通过查找合适的hbase:meata区域获取用户空间区域所在节点和位置，接着就可以交互。
[hbase组件间交互](http://www.voidcn.com/article/p-ncmwzrge-ng.html)   
到达regionserever的写操作首先追加到"提交日志",然后加入内存中的memstore，如果memstore满，内容会被刷入文件系统。XXX
扫描器：和传统数据库中游标或java中迭代器不同在于使用后需要关闭。ResultScanner  缓存。  
**HBase和RDBMS的比较**：  
分布式，面向列的数据存储系统，在HDFS上提供随机读写解决Hadoop不能解决的问题。自底层设计解决可伸缩性问题，表可以很高很宽。表的模式是物理存储的直接反映，
使系统有可能提供高效的数据结构的序列化，存储和检索。标准的RDBMS遵循codd规则，是模式固定，面向行的数据库具有ACID性质和复杂的SQL查询处理引擎，可以非常容易建立二级索引执行复杂的
内连接和外连接，执行计数，排序分组或者对表行列中的数据分页。
**HBase特性：**
>* 行是顺序存储，每行的列也是，不存在真正的索引，所以没有这方面的顾虑
>* 自动分区
>* 线性扩展和对于新节点的自动处理
>* 容错
>* 普通商用硬件支持
>* 批处理，集成mr功能

**HBase常见问题**
>* 文件描述符用完
>* datanode线程用完，现在hadoop2一般不会出现这个问题