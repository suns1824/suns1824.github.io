##预热知识点:  
 [快速失败与安全失败](https://blog.csdn.net/u010889616/article/details/79954413)
 
## HashMap和ConcurrentHashMap
[源码剖析](http://www.importnew.com/28263.html)
ps: ConcurrentHashMap的get()方法简单高效，整个过程不需要加锁，除非读到的值为空才会加锁重读 -------- 因为它的get方法里将要使用的共享变量都定义成volatile类型：如用于统计当前
segment大小的count字段和用于存储值的HashEntry的value。这里可以结合一下无锁编程理解。
## ConcurrentLinkedQueue
CAS实现非阻塞算法实现的线程安全的队列

## List
[为什么没有线程安全的Arraylist](http://ifeve.com/why-is-there-not-concurrent-arraylist-in-java-util-concurrent-package/)     
[CopyOnWriteArrayList和Collections.synchronizedList比较](https://www.cnblogs.com/yan456jie/p/5369366.html)  
Vector是线程安全的：
```java
public class Test {
    public static void main(String[] args){
      List<String> data = new Vector<>();
      CountDownLatch latch = new  CountDownLatch(100);
      for(int i = 0; i < 100; i++) {
          SampleTask task = new SimpleTask(data, latch);
          new Thread(task).start();
      }
      try{
          latch.await();
      } catch (InterruptedException e) {
          e.printStackTrace();
      }
      System.out.println(data.size());
    }
}

class SampleTask implements Runnable {
    CountDownLatch latch;
    List<String> data;
    public SampleTask(List<String> data, CountDownLatch latch) {
        this.data = data;
        this.latch = latch;
    }
    
    @Override
    public void run() {
        for(int i = 0; i < 100; i++) {
            data.add("1");
        }
        latch.countDown();
    }
}
```
如果使用ArrayList不一定输出10000，可以在run方法里使用**synchronized(data)**。

关于List，有这么一个方法：
```text
List list= Collections.synchronizedList(new LinkedList());
```
需要注意的是，迭代这个list时需要**synchronized(list)**,因为iterator方法没有实现同步。Collections类中有一个SynchronizedCollection类，这个类有一个
成员变量mutex，除了迭代和stream流外的绝大多数操作都使用了synchronized(mutex)实现了线程同步。SynchronizedList继承了这个类。


## 阻塞队列