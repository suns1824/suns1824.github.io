
## Linux网络I/O模型
Linux内核将所有的外部设备都看作一个文件来操作，对一个文件的读写操作会调用内核提供的系统命令，返回一个file descriptor。而对一个socket的读写也会有相应的描述符，成为socketfd，描述符就是一个数字，它
指向内核中的一个结构体(文件路径，数据区等属性)。
Unix网络编程中5种I/O模型：
>* 阻塞IO模型： 应用进程发起recvfrom系统调用，从这时候开始，一直到内核空间准备好数据并把数据复制到用户空间，进程都是被阻塞的。
>* 非阻塞IO模型： recvfrom XXX，如果缓冲区没有数据的话，就直接返回一个EWOULDBLOCK错误，进程不断轮询检查这个状态,看内核是否有数据到来。
>* IO复用模型： 进程将一个或多个fd传递给select/poll系统调用（此时进程阻塞在select操作上），select/poll/epoll函数会不断轮询查询fd状态是否处于就绪状态（有数据到来），是则通知用户进程。
>* 信号驱动IO模型：开启socket信号驱动IO功能，并通过系统调用sigaction执行一个信号处理函数，当数据准备就绪时，为该进程生成一个sigio信号（在此之前进程不被阻塞），通过信号回调通知进程调用recvfrom读取数据（阻塞）并处理数据。
>* 异步IO：告知内核启动某个操作，并让内核在整个操作完成后通知我们。
 
理解5种IO模型的关键之一：理解何时阻塞何时正常执行  
[相关链接](https://blog.csdn.net/fgf00/article/details/52793739)   
[阻塞同步](https://blog.csdn.net/yyxyong/article/details/62894064)  

### I/O多路复用
I/O编程过程种，当要同时处理多个客户端接入请求时，可以利用多线程或者I/O多路复用技术。后者通过把多个I/O的阻塞复用到同一个select的阻塞上，从而使得系统可以在单线程的情况下同时处理多个客户端请求。与传统的多线程模型相比，I/O多路复用的最大优势是系统开销小。  
select和poll是顺序查询fd状态是否就绪，导致任意时刻只有少部分socket是“活跃”的，都会线性扫描全部fd，效率线性下降。epoll则只会对“活跃”的socket进行操作--这是因为在内核的实现中，epoll基于事件驱动来代替顺序查询，它是根据每个fd上面
的callback函数实现的，只有“活跃”的socket才会去主动调用callback函数，其他idle状态的socket则不会（伪AIO）。同时没有最大fd的数量限制，epoll同时支持边缘触发和水平触发。同时，无论select，poll还是epoll都需要把fd消息通知给用户空间，需要内存复制，而epoll通过内核和用户空间mmap同一块内存实现的。   

## NIO入门
### 传统BIO（同步阻塞，代码参考xxx）
一个线程处理一个客户端连接，伪异步则是通过线程池的方式实现了N:M的比例，但是依旧没有改变同步阻塞的本质缺点。   
首先查看InputStream的read(bytep[])方法：**this method blocks until input data is available, end of file is detected, or an exception is thrown**,这意味着当对方发送请求或者应答消息比较缓慢或网络不好时读取输入流的同学现场将会被长时间阻塞，在此期间，
其他接入消息只能在消息队列中排队。   
同样查看OutputStream的write(byte[])方法，这个方法写输出流的时候将会被阻塞，直到所有要发送的字节全部写入完毕或者发生异常。  
**延申**：当消息的接收方处理缓慢的时候，将不能及时从TCP缓冲区读取数据，这将会导致发送方的TCPwindow size不断减少直到为0，双方处于Keep-alive状态，消息发送方间不再向TCP缓冲区写入消息，这是如果采用同步阻塞IO，write将会XXX。
所以可以**总结**一下BIO的各种缺点。   

### NIO编程
NIO就是非阻塞IO，与Socket和ServerSocket类相对应，NIO也提供了SocketChannel和ServerSocketChannel两种不同的套接字通道实现，这两种新增的通道都支持阻塞和非阻塞两种模式。  
#### 缓冲区Buffer
Buffer是一个对象，包含一些要写入或者读出的数据。在NIO库中，所有数据都是用缓冲区处理的。缓冲区是实质上是一个数组，提供了对数据的结构化访问和维护读写位置等信息。
#### 通道Channel
通道与流不同之处在于通道是双向的。Channel全双工，所以更好地映射底层操作系统的API（UNIX网络编程模型中，底层操作系统的通道都是全双工的，同时支持读写操作）。
#### 多路复用器Selector
多路复用器不断轮询注册在其上的Channel(epoll机制使得数量没有限制)，如果某个Channel上面发生读/写时间，这个Channel就处于就绪状态，会被Selcetor轮询出来，然后通过SelectionKey可以获取就绪Channel的集合，进行后续的I/O操作。   
**源码见SparkPro工程netty包**    
总结一下NIO编程优点：
>* 客户端发起的连接操作是异步的，可以通过在多路复用器注册OP_CONNECT等待后续结果，不需要像之前的客户端那样被同步阻塞
>* SocketChannel的读写操作都是异步的，如果没有可读写的数据不会同步等待，直接返回，这样IO通信线程就可以处理其他的链路，不需要同步等待这个链路可用。
>* 线程模型的优化，Selector在Linux等主流系统上通过epoll实现，没有连接句柄的限制，意味着一个Selector线程可以同时处理成千上万个客户端连接，性能不会随着客户端的增加而下降，适合做高性能高负载的网络服务器。

### AIO编程
NIO2.0引入了新的异步通道的概念，并提供了异步文件通道和异步套接字通道的实现。异步通道提供以下两种方式获取操作结果：
> * java.util.concurrent.Future类来表示异步操作的结果
> * 在执行异步操作的时候传入一个java.nio.channel

NIO2.0的异步套接字通道是真正的异步非阻塞I/O，对应于UNIX网络编程的事件驱动I/O(AIO)。它不需要通过Selector对注册的通道进行轮询操作即可实现异步读写，从而简化了NIO的编程模型。   
**代码在XXX**

延申： 编解码技术，序列化和ByteBuffer的二进制编解码技术对比， Protobuf，[边缘触发和水平触发](https://blog.csdn.net/lihao21/article/details/67631516)。