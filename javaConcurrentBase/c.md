## 线程池
好处：
>* 重复利用已经创建的线程降低线程创建销毁的开销 
>* 任务到达时可以不需要等到线程创建就能立即执行
>* 线程池对线程统一分配，调优和监控--提高线程的可管理性

### 实现原理
#### 处理流程
当一个任务被提交到线程池的时候：
1. 线程池判断核心线程池里的线程是否都在执行任务。如果不是，new一个新的工作线程来执行任务(需要获取全局锁)；如果是，进入下一个流程   
2. 线程池判断工作队列是否已满，如果没有满，则将新提交的任务存储在这个工作队列中。如果满了，进入下一个流程
3. 线程池判断线程池的线程是否都处于工作状态，如果没有，创建一个新的工作线程来执行任务(需要获取全局锁)；如果满了，则交给饱和策略来执行这个任务   

这也反应在ThreadPoolExecutor.execute(Runnable)的源码中。  
还有关于工作线程的**职责**有两点：执行当前任务和循环从BlockingQueue中获取任务来执行   
**理解概念**：corePoolSize，BlockingQueue，maximumPoolSize     

#### 线程池的使用
**创建**：    
```text
new ThreadPoolExecutor(corePoolSize, maximumPoolSize, keepAliveTime, milliseconds, runnableTaskQueue, handler)
```
handler -- RejectedExecutionHandler(饱和策略)：默认直接抛出异常，当然还有其他策略：脑补。   
**向线程池提交任务**：    
可以使用execute()方法和submit()方法
execute()用于提交没需要返回值的方法，所以无法判断任务是否被线程池执行成功：
```text
threadsPool.execute(new Runnable() {
  run() {
    //xxx
  }
});
```
submit()方法用于提交需要返回值的任务。线程池会返回一个future类型的对象，通过这个future对象可以判断任务是否执行成功，并且可以通过future的get()方法来获取返回值，
get()方法会阻塞当前线程直到任务完成，get(long timeout, TimeUnit unit)则会阻塞当前线程一段时间后返回，此时任务可能没有执行完成：
```text
Future<Object> future = executor.submit(task);
try {
  Object s = future.get();
} catch (InterruptedExceptino e) {
  
} catch (ExecutionException e) {

} finally {
   executor.shutdown();
}
```
**关闭线程池**
可以通过线程池 shutdown或者shutdownNow方法来关闭线程池。原理：遍历线程池中的工作线程，然后逐个调用线程的interrupt方法来中断线程，所以无法响应中断的任务可能永远无法终止。   
**记住shutdown和shutdownnow的区别**  
shutDown:将线程池的状态设置为SHUTDOWN状态，然后中断所有没有正在执行任务的线程；   
shutDownNow：将线程池状态设为STOP，然后尝试关闭所有正在执行或暂停任务的线程，并返回正在等待执行任务的列表。  
当所有任务都已经关闭，线程池关闭成功，isTerminated返回true。
**合理配置线程池**
考虑任务性质(IO or CPU or 混合)  && 任务优先级   &&  任务执行时间  && 任务的依赖性(是否依赖其他系统资源)：  
>* CPU密集型  N+1 个线程    
>* IO密集型   2^N 个线程   
>* 混合型    将任务拆分成两个任务(两者执行时间不要相差太大)   
>* 优先级不同的任务可以使用优先级队列(PriorityBlockingQueue)，不过xxx   
>* 执行时间不同的任务可以交给不同的线程池处理或者使用优先队列  
>* 依赖数据库连接池的任务，因为线程提交SQL后需要等待返回结果，CPU空闲时间长，所以线程数尽量设置大些   
>* 建议使用有界队列

**线程池监控**
重写beforeExecute,afterExecute,terminated方法监控线程池的属性  

## Executor框架
Java线程既是工作单元也是执行机制。JDK5开始，把工作单元与执行机制分开来，工作单元包括Runnable和Callable，而执行机制有Executor框架提供。