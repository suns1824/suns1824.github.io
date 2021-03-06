 ### 运行机制
 Yarn通过两类长期运行的守护进程提供自己的核心服务：
 >* 管理集群上资源使用的资源管理器(resource manager)
 >* 运行在集群中所有节点上能够启动和监控容器(container)的节点管理器(node manager)
 
 容器用于执行特定应用程序的进程，每个容器都有资源限制。一个容器可以是一个Unix进程，也可以是一个Linux cgroup(对进程使用系统资源进行控制)。   
 Yarn上运行应用的流程大致为：客户端(Application client)联系资源管理器，要求它运行一个application master进程。然后，资源管理器找到一个能够在容器中启动application master的节点管理器。     
 YARN本身不会为应用的各个部分(客户端、master、进程)彼此通信提供任何手段，依赖于Hadoop的RPC层。  
 资源分配尽量本地化。
 
 ### YARN中的调度
 #### 调度选项
 YARN中由三种调度器可用：
 >* FIFO调度器
 >* 容量调度器
 >* 公平调度器
 
 使用容量调度器时有一个专门的队列保证小作业一提交就可以启动，公平调度器会在所有运行的作业之间动态平衡资源。
 
 [yarn三大模块](https://www.cnblogs.com/BYRans/p/5513991.html)   
 [spark的yarn-client模式和yarn-cluster模式](https://www.iteblog.com/archives/1223.html)