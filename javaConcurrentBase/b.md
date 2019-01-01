
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
ps:当一个Java虚拟机中不存在非Daemon线程时，Java虚拟机会退出。但是Daemon线程中finally块不一定会执行。 

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
中断表示一个运行中的线程是否被其他线程进行了中断操作。多查阅资料学习。
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

写一个独占锁（同一时刻只能有一个线程获取到锁，其他获取锁的线程只能处于同步队列中等待）：
```java
class Mutex implements Lock {
    private static class Sync extends AbstractQueuedSynchronizer {
        // 是否处于占用状态
        protected boolean isHeldExclusively() {
            return getState() == 1;
        }
        // 状态为0时获取锁
        public boolean tryAcquire(int acquires) {
            if (compareAndSetState(0,1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }
        //释放锁，将状态设置为0
        protected boolean tryRelease(int releases) {
                if (getState() == 0) throw new IllegalMonitorStateException();
                setExclusiceOwnerThread(null);
                setState(0);
                return true;
        }
        Condition newCondition() {
            return new ConditionObject();
        }
    }
    
    private final Sync sync = new Sync();
    public void lock() {
        sync.acquire(1);
    }
    public boolean tryLock() {
        return sync.acquire(1);
    }
    public void unLock() {
        sync.release(1);
    }
    public Condition newCondition() {
        reuturn sync.newCondition();
    }
    public boolean isLocked() {
        return sync.isHeldExclusively();
    }
    public boolean hasQueuedThreads() {
        return sync.hasQueuedThreads();
    }
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }
    public boolean tryLock(long timeout, TimeUnit unit) throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }
}
```
### 队列同步器的实现分析
#### 同步队列 
同步器拥有首节点和尾节点，加入同步队列的操作需要保证线程安全，因此，同步器提供了一个基于CAS的设置尾节点的方法：**compareAndSetTail(Node expect, Node update)**。由于只有一个线程能够获取到同步状态，因此设置头节点的方法不需要CAS。
### 独占式同步状态获取与释放
```java
//该方法对中断不敏感
public final void acquire(int arg) {
    if( !tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) 
        selfInterrupt();
}
```
addWaiter方法使用之前所说CAS确保节点被线程安全添加（试想如果多个线程调用tryAcquire方法获取同步状态失败而并发地添加到LinkedList上，xxx）。
```java
private Node addWaiter(Node node) {
    Node node = new Node(Thread.currentThread(), mode);
    Node prev = tail;
    if (prev != null) {
        node.prev = prev;
        if (compareAndSetTail(prev, node)) {
            prev.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
```
addWaiter中调用了enq(final Node)方法,“死循环”方式添加尾节点，一目了然：
```java
private Node enq(final Node node) {
    for(;;) {
        Node t = tail;
        if (t == null) {
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```
节点进入同步队列后，就进入了一个自旋的过程，具体参考acquireQueued方法。
释放操作：唤醒头节点的后继节点，使用了LockSupport。
#### 共享式同步状态获取与释放

```java
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0) {  //共享式获取的自旋过程中，成功获取同步状态并退出自旋的条件就是xxx
        doAcquireShared(arg); //判断前驱结点是否为头节点，是则获取同步状态。
    }
}
```
同理，释放xxx，和独占式的主要区别：tryRelaeseShared(int arg)必须确保同步状态线程安全释放（循环和CAS），因为释放同步状态的操作会来自于多个线程。
#### 独占式超时获取同步状态
超时获取同步状态过程可以被视为响应中断获取同步状态过程的增强版，doAcquireNanos方法在支持响应中断的基础上，增加了超时获取的特性。
## 重入锁
ReentrantLock，支持重进入的锁，表示该锁能够支持一个线程对资源的重复加锁。  
synchronized关键字隐式支持重进入，ReentrantLock在调用lock方法时，已经获取到锁的线程，能偶再次调用lock()方法获取锁而不被阻塞。   
ReentrantLock分为公平锁和非公平锁两种，具体直接看源码。两者实现的区别。非公平锁开销更小（非公平锁里，当一个线程请求锁的时候，只要获取了同步状态即成功获取锁，所以刚释放锁的线程获取同步状态的几率大，线程切换少）。
## 读写锁（出现之前用等待通知的方式解决）
读写锁(ReentrantReadWriteLock)维护了一对锁：一个读锁和一个写锁，通过分离读锁和写锁，使得并发性相比一般的排他锁有了很大提升。
特点：
>* 支持非公平和公平性获取锁
>* 支持重进入，读线程在获取读锁后能再次获取读锁，写线程获取写锁后能再次获取写锁和读锁
>* 锁降级，遵循获取写锁，获取读锁再释放写锁的次序，写锁能够降级为读锁

ps：ReentrantReadWriteLock的getReadHoldCount返回当前线程获取读锁的次数，通过ThreadLocal保存。
**思考**：如何让HashMap做到线程安全？ ------> 借助读写锁   
### 读写锁的实现分析(同样依赖自定义同步器)
ReentrantLock中同步状态表示锁被一个线程重复获取的次数，读写锁的自定义同步器需要在同步状态上维护多个读线程和一个写线程的状态（按位切割，高16位表示读，低16位表示写）。
写锁和读锁源码seeJDK。   
**锁降级**：如果当前写出拥有写锁，然后将其释放，最后再获取读锁，这种分段完成的过程不能称为锁降级。锁降级指的是：把持住写锁，再获取到读锁，然后释放写锁的过程。   
**思考**：锁降级中读锁的获取是否有必要？ --------- 有必要：为了保证数据的可见性，如果当前线程不获取读锁就直接释放写锁，那么另外一个线程获取写锁并修改了数据，那么当前线程无法感知线程X的数据更新。如果当前线程获取读锁，那么线程T将会被阻塞，
知道当前线程使用数据并释放读锁之后，线程X才能获取写锁更新数据。XXX不支持锁升级的原因也是保证数据可见性：如果读锁被多个线程获取，其中一个线程成功获取了写锁并更新了数据，其更新对其他获取到读锁的线程不可见。
## LockSupport工具
当需要阻塞或者唤醒一个线程的时候，都会使用LockSupport工具类来完成相应工作。LockSupport定义了一组以park开头的方法来阻塞当前线程，以及unpark(Thread thread)方法来唤醒一个被阻塞的线程。LockSupport也成为了
构建同步组件的基础工具。  
## Condition接口
Condition与Object的监视器方法对比：XXX。   
Condition定义了等待/通知两种类型的方法，当前线程调用这些方法时，需要提前获取到Condition对象关联的锁。Condition对象由Lock对象创建。    
### 实现分析 
ConditionObject(实现了Condition接口)是同步器AbstractQueuedSynchronizer的内部类，因为Condition操作需要获取象关联的锁，所以作为同步器内部类较为合理。   
一个Condition包含一个等待队列。在Object的监视器模型上，一个对象拥有一个同步队列和一个等待队列；而并发包里的Lock（更确切地说是同步器）拥有一个同步队列和多个等待队列(一个同步器可以has多个Condition实例)。   
Condition拥有首尾节点的引用，新增节点只需将原有的尾节点指向它，并且更新尾节点即可（不需要CAS保证，因为XXX）。   
**参考一下Condition的await方法和signal方法的源码理解原理**

# Java并发容器和框架
## Java中的阻塞队列
阻塞队列是一个支持两个附加操作的队列(支持阻塞的插入方法和支持阻塞的移除方法)。  联想生产者消费者的场景。
ArrayBlockingQueue原理：使用了通知模式来实现，当生产者往满的队列里添加元素时会阻塞住生产者，当消费者消费了一个队列的元素后，会通知生产者当前队列可用(Conditon来实现)。  
延申：看下Unsafe.park等方法，都是native方法。
## Fork/Join框架
一个并行执行任务的框架，是一个把大任务分割(fork)成若干个小任务，最终汇总(join)每个小任务结果后得到大任务结果的框架。   
**延申**：工作窃取算法：指某个线程从其他队列里窃取任务来执行,为例减少窃取任务线程和被窃取线程之间的竞争，通常使用双端队列，被窃取线程永远从自己的等待任务队列的头部拿任务执行，而XXX。
# Java中的原子操作类
java.util.concurrent.atomic包的类基本都是使用Unsafe实现的包装类。
运用场景，以AtomicInteger为例：[AtomicInteger](https://haininghacker-foxmail-com.iteye.com/blog/1401346)

# Java中的并发工具类
## CountDownLatch
假设一个需求：解析一个Excel里多个sheet的数据，考虑使用多线程，每个线程解析一个sheet里的数据，所有sheet解析完成后，程序提示完成。最简单的做法是join()方法：
```text
main{
   thread1.start();
   thread2.start();
   thread1.join();
   thread2.join();
   System.out.println("ok");
}
```
其原理是不停检查join线程是否存活，如果join线程存活则让当前线程永远等待：
```text
while(isAive()) {
  wait(0)  //永远等待下去
}
```
直到join线程终止后，线程的this.notifyAll()方法会被调用。   
JDK1.5提供了CountDownLatch，它的构造函数接收一个int类型的参数，如果想要等待N个点（可以是N个线程，也可以是1个线程中的N个步骤）完成，那就传入N。方法：countDown调用N就会-1，CountDownLatch的await方法会阻塞当前线程，直到N变为0。   
在[SparkPro的netty包里AIO代码](https://github.com/suns1824/SparkPro/blob/master/src/main/java/com/happy/netty/aio/AsyncTimeClientHandler.java)中有CountDownLatch的例子。  
## CyclicBarrier
让一组线程到达一个屏障(同步点)时被阻塞直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续运行。每个线程调用await方法来告诉CyclicBarrier已经到达屏障。   
**理解两个构造函数**   
两者区别：CountDownLatch计数器只能使用一次，而CyclicBarrier的计数器可以使用reset()方法重置，当然还提供了其他很多方法。  
## Semaphore
用来控制同时访问特定资源的线程数量，它通过协调各个线程，保证合理使用资源。**理解acquire和release方法**
## Exchanger
用于进行线程间的数据交换，它提供一个同步点，在这个同步点上，两个线程可以交换彼此的数据。
