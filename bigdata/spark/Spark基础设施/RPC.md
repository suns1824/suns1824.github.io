Spark中很多地方都涉及网络通信：
>* Spark各个组件间的消息互通；
>* 用户文件于Jar包的上传；
>* 节点间的Shuffle；
>* Block数据的复制与备份。

Spark1.x.x版本中，用户文件与Jar包的上传使用的是由Jetty实现的HttpFileServer，但是在Spark2.0.0版本中它被废弃了，现在使用的是基于Spark内置RPC框架的NettyStreamManager。节点间的Shuffle过程和
Block数据的复制与备份依然沿用了Netty。