
# Java并发基础
## 线程
现代操作系统调度的最小单元，拥有各自的计数器，堆栈和局部变量等属性，并且能够访问共享的内存变量。  
Java线程在运行的生命周期中存在6种状态：
>* NEW
>* RUNNABLE
>* BLOCKED
>* WAITING
>* TIME_WAITING
>* TERMINATED

大体流程：  
**NEW** -------Thread.start()------ **RUNNABLE（运行中和就绪状态）** ------执行完成------ **TERMIATED**    
Runnable<--->Waiting可以通过如下方法：
>* Object.wait()| Object.notify()
>* Object.join()| Object.notifyAll()
>* LockSupport.park()| LockSupport.unpark(Thread)

Runnable<--->Timed_Waiting 类似方法  
Runnable--->Blocked：等待进入synchronized块/方法，反之是获取到锁。  

### Daemon线程
一种支持性线程，被用作程序中后台调度以及支持性工作。 
ps:当一个Java虚拟机种不存在非Daemon线程时，Java虚拟机会退出。但是Daemon线程中finally块不一定会执行。 

## 启动和终止线程
### 构造线程
```java
java.lang.Thread.init(ThreadGroup, Runnable, String name, long stackSize, AccessControlContext add) 
  //一个新构造的线程对象是由其parent线程来进行空间分配的，而child线程继承了parent是否为Daemon，优先级和加载资源的contextClassLoader以及
  //可能的ThreadLocal，同时还会分配一个Id。
}
```
### 启动线程
调用start()方法，含义：当前(parent)线程同步告知JVM，只要线程规划器空闲，应当立即调用start()方法的线程。  
### 中断
中断表示一个运行中的线程是否被其他线程进行了中断操作。
xxx: 关于中断标识位被清除的概念。  
### 安全地终止线程
一种是interrupt()方法，还有一种通过一个boolean变量的方式:xxx。   

## 线程间通信
>* volatile和synchronized关键字(顺便理解一下对象监视器和同步队列与线程的关系)
>* 等待/通知机制(notify和wait方法，调用wait方法的线程进入waiting状态，只有等待另外线程通知或者自己被中断才会返回，该方法会**释放对象的锁**，理解监视器，同步队列，等待队列和执行线程的关系)
>* 管道输入/输出流(用于线程间数据传输，传输媒介为内存，面向字节：PipedOutputStream,PipedInputStream;面向字符：PipedReader和PipedWriter)
>* Thread.join(涉及到等待/通知机制)
>* ThreadLocal(一个以ThreadLocal对象为键，任意对象为值的存储结构，这个结构被附着在线程上)

等待/通知机制的经典范式：
```java
synchronized(对象) {
    while(条件不满足) {
        对象.wait();
    }
    ...
}
```

# Java中的锁
## Lock接口
Lock接口出现之前，Java通过synchronized实现锁功能。相比于synchronized，它缺少了隐式获取释放锁的便捷性，但是却拥有了锁获取释放的可操作性，可中断地获取锁以及超时获取锁
等synchronized锁不具备的同步特性：
>* 当前线程尝试获取锁，如果这一时刻锁没有被其他线程获取到，则成功获取并持有锁（非阻塞地获取锁）
>* 与synchronized不同，获取到锁的线程能够响应中断 
>* 超时获取锁

熟悉Lock的API。
## 队列同步器
AbstractQueuedSynnchronizer是用来构建锁或者其他同步组件的基础框架，它使用了一个int成员变量表示同步状态，通过内置的FIFO队列来完成资源获取线程的排队工作。
同步器的设计是基于模板方法模式的，使用者需要继承同步器并重写同步器指定方法，随后将同步器组合在自定义同步组建的实现中，并调用同步器提供的模板方法,而这些模板方法将会调用使用者重写等待方法(参考这个类源码和自定义同步器例子理解)。
同步器里一些概念：访问/修改同步状态的方法，可重写的方法，模板方法
模板方法(3类：独占/共享式获取与释放同步状态，查询同步队列中等待线程情况)：xxx。
### 队列同步器的实现分析
## 可重入锁
## 读写锁
## LockSupport
## Condition接口

# Java并发容器和框架

# Java中的原子操作类

# Java中的并发工具类

# 线程池和Executor框架