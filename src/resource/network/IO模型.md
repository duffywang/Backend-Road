## IO模型

###引子
一排水壶再烧开水，
BIO的工作为直到这个水壶烧开，才去处理下一个水壶，在等待水壶烧开的时间段什么都没有做
NIO的工作让一个线程不断地轮询每个水壶的状态，看看水壶是否烧开，如果烧开了进行处理
AIO的工作是在每一个水壶上面装了一个开关，水烧开自动通知我水烧开了


区分同步或异步（synchronous/asynchronous）。
简单来说，同步是一种可靠的有序运行机制，当我们进行同步操作时，后续的任务是等待当前调用返回，才会进行下一步；
而异步则相反，其他任务不需要等待当前调用返回，通常依靠事件、回调等机制来实现任务间次序关系。

区分阻塞与非阻塞（blocking/non-blocking）。
在进行阻塞操作时，当前线程会处于阻塞状态，无法从事其他任务，只有当条件就绪才能继续，比如ServerSocket新连接建立完毕，或数据读取、写入操作完成；
而非阻塞则是不管IO操作是否结束，直接返回，相应操作在后台继续处理。
不能一概而论认为同步或阻塞就是低效，具体需要看应用和系统特征。

操作系统提供的标准网络IO有以下成本
- 系统调用机制产生的开销
- 数据多次拷贝的开销（数据总是先写到操作系统缓存再到用户传入的内存）
- 因为没有数据而阻塞，产生调度重新获得执行权，产生时间成本。
- 线程空间成本和时间成本

### BIO

同步阻塞 1:1
在活动连接数不是特别高（小于单机1000）的情况下，这种模型是比较不错的，可以让每一个连接专注于自己的 I/O 并且编程模型简单，也不用过多考虑系统的过载、限流等问题。
线程池本身就是一个天然的漏斗，可以缓冲一些系统处理不了的连接或请求。但是，当面对十万甚至百万级连接的时候，传统的 BIO 模型是无能为力的。因此，我们需要一种更高效的 I/O 处理模型来应对更高的并发量。
EndPoint可以理解为是Tomcat的端点类，用于处理Socket连接，BIO对应的EndPoint是JIOEndPoint，当JIOEndPoint启动时，会将Acceptor线程池和工作线程池启动起来。Tomcat的BIO模式中有两类线程：

1. Accept线程

- 用于接收Socket连接，如果没有连接请求，该线程会一直serversocket.accept阻塞住。当有连接请求时，首先为socekt设置Connector配置的属性，然后在工作线程池中找出空闲的线程，将接力棒传递给工作线程池，如果worker线程池暂时没有空闲线程，则Acceptor线程会阻塞等待。

- 使用LimitLatch限制最大连接数。

2.工作线程

工作线程池中处理的是SocketProcessor任务，Acceptor传过来的Socket经过封装转换为SocketProcessor任务，提交到工作线程池中执行。

当有新的连接请求时，Accept线程判断还没有超出最大连接数，则将socket封装为SocketProcessor任务放到工作线程池执行（如果工作线程池没有空闲线程，Accept线程会阻塞等待），SocketProcessor会调用Http11Processor.process( )，process( )方法中会触发操作系统的read操作，并一直阻塞等待数据就绪，期间工作线程被挂起不能做其他操作。数据拷贝完成后，工作线程才将数据进行封装并交由容器处理。

BIO模式的好处是可以让一个请求在一个线程中执行，理解成本和实现成本都比较低，如果当前请求没有复杂的I/O操作，业务逻辑也非常简单，很多时候是可以在一个CPU时间片内就处理完成，这种情况下BIO的处理效率是非常高的。但是这种模式存在两个问题：

1. Accept线程工作效率低

目前大多数的HTTP请求使用的都是长连接（HTTP/1.1 默认keep-alive=true），长连接意味着：一个TCP的socket在当前请求结束后，如果没有新的请求到来，socket不会立即释放，而是等timeout后再释放。如果使用BIO，读取socket并交给工作线程处理这个过程是阻塞的（只有找到空闲工作线程，Accept才能顺利将该请求转交给工作线程，否则会一直等待），也就意味着socket在处理当前请求时，当前的Accept线程是一种被占用的状态，无法释放去处理其他的请求，因此在大并发场景下，Accept线程就要设置的非常高，多线程会造成竞争加剧，加上多线程调度都会耗费较多的CPU资源。

2. 网络I/O的阻塞等待导致工作线程的效率低

在Tomcat处理用户请求时，数据可能还没有真正到达网卡，而BIO模式在处理网络I/O时是阻塞读取的，需要等到数据真实到达网卡，由操作系统将数据从网卡拷贝到内核，再由用户线程触发读操作，然后操作系统将内核数据拷贝至用户空间缓冲区，工作线程需要阻塞等待这些过程完毕，中间不能做任何事，工作线程的效率被大大降低，同时工作线程还需要进行线程状态切换，会浪费CPU资源。

由上分析可以看出无论是Accept线程还是工作线程，效率都较低，因此BIO是一种效率较低的模式。
### NIO
同步非阻塞 N:1
适用于高负载、高并发的（网络）应用
采取Reactor模式，或者说是Observer模式为我们监察I/O端口，其中将Channel注册到Selector上，指定感兴趣的具体事件，Selector类似于观察者，

Tomcat的NIO是基于Java的NIO，NIO对应的EndPoint是NIOEndPoint，当NIOEndPoint启动时，会启动Accept线程池、Poller线程及工作线程池，并创建Poller，Poller中会维护一个PollerEvent的队列。Tomcat的NIO模式中有三类线程：

1. Accept线程（与BIO的Accept线程一致）

2. Poller线程池

Poller中维护的是Selector对象，Selector有两个作用：一是将从Acceptor传过来的NioChannel进行感兴趣事件的NIO注册；二是轮询感兴趣的事件是否发生，感兴趣的事件指的是数据是否就绪。当轮询到事件发生则将接力棒交给工作线程处理。

3. 工作线程

工作线程会进行Http协议的解析，然后将解析出来的内容包装成Request、Reponse对象，传递给CoyoteAdapter，最终执行到业务中。

当有新的连接请求时，Accept线程判断还没有超出最大连接数，则将socket封装为NioChannel，并把这个NioChannel注册到Poller中。注册过程：首先将NioChannel封装成NioSocketWrapper，并注册一个PollerEvent事件（该事件持有NioSocketWrapper对象）到一个缓冲队列中。Poller线程在NIOEndPoint启动时就已经启动，Poller线程会轮询，使用select方式选取已经I/O就绪（指数据已经到达内核空间，但还没有到达用户空间）的文件描述符，获取它持有的NioSocketWrapper对象，然后调用NIOEndPoint的processKey方法对其进行处理，该方法会将NioSocketWrapper封装为一个SocketProcessor任务交由工作线程池处理。在SocketProcessor任务会调用Http11Processor.process( )，Http11Processor中会触发操作系统的read操作，并阻塞等待操作系统将数据从内核空间拷贝至用户空间，期间工作线程被挂起不能做其他操作。数据拷贝完成后，工作线程将数据进行封装并交由容器处理。

NIO相比BIO多了Poller线程池，是非常典型的 “生产者-消费者” 模式，相较于BIO，NIO提升了Accept线程和工作线程的工作效率。

1. 提升了Accept线程工作效率

Accept线程只用向PollerEvent缓冲区中生产请求事件，Poller线程负责轮询这些事件，当有可读事件发生时，再交由工作线程处理。虽然从流程上来说是多了一道手续，但是由于多了一个事件缓冲区的设计，Accept线程只用生产事件，不直接与工作线程进行交互，不用阻塞等待工作线程空闲，因此Accept线程的利用率得到了大大的提升。

2. 提升了网络I/O的效率

网络I/O是非阻塞的，Poller线程轮询到可读事件时才触发Http11Processor去处理，此时数据已经到达内核缓冲区，用户线程读取数据时只用触发操作系统的read操作将内核数据拷贝至用户空间，是典型的同步非阻塞模型。
### AIO
异步、非阻塞 M:0
不需要启动额外的IO，被动回调

NIO2对应的EndPoint是NIO2EndPoint。Tomcat的AIO模式中有两类线程：

1. Accept线程（与BIO的Accept线程一致）

2. 工作线程池

- 触发read信号；

- 注册回调函数，回调函数中执行业务逻辑。

Acceptor 接收新的连接后，得到一个 AsynchronousSocketChannel，Acceptor 把AsynchronousSocketChannel封装成一个 Nio2SocketWrapper，并创建一个SocketProcessor任务类交给工作线程池处理，该任务中会触发read操作，同时注册回调类 readCompletionHandler，因为此时数据没读到，Http11Processor把当前的Nio2SocketWrapper标记为数据不完整，并维护了一个数据不完整的Nio2SocketWrapper列表维护连接状态，同时将该工作线程回收。当数据到达内核后，操作系统会主动将数据从内核拷贝至Http11Processor提前在用户空间指定的Buffer里，此时提前注册的回调类 readCompletionHandler 被调用，在这个回调处理方法里会重新创建一个新的 SocketProcessor 任务来继续处理这个连接，而这个新的 SocketProcessor 任务类持有原来那个 Nio2SocketWrapper，此时数据已经可读（在用户空间），新的工作线程继续执行后续的业务逻辑。

可以看到Tomcat的AIO模式中，“触发数据由内核空间拷贝至用户空间”这个动作完全由操作系统主动来完成，工作线程完全不用阻塞等待数据拷贝的完成，线程资源可以得到更有效的利用。


三种模式不同点主要体现在两点：

1. 数据在拷贝过程中，容器的工作进程的状态是怎样的？

BIO模式在操作系统将数据拷贝至用户空间缓冲区时，用户态的工作线程一直处于挂起等待状态，是阻塞方式。NIO、AIO在数据拷贝期间仍然可做其他事。

2. 数据在内核空间就绪后用什么方式拷贝至用户空间？

BIO、NIO均是采用进程主动询问的方式得知数据就绪，并由用户线程触发read操作，转由操作系统进行数据拷贝。而AIO则通过回调的方式，在内核数据就绪后，由操作系统主动将数据拷贝至用户线程提前指定的用户空间Buffer，是典型的异步方式。
## Tomcat 


## Servlet

异步监听器AsyncListener
```java
public interface AsyncListener extends EventListener {
        public void onComplete(AsyncEvent event) throws IOException;
        
        public void onTimeout(AsyncEvent event) throws IOException;

        public void onError(AsyncEvent event) throws IOException;
        
        public void onStartAsync(AsyncEvent event) throws IOException;     
        
}

```

读监听器ReadListener
```java
    /**
     * When an instance of the <code>ReadListener</code> is registered with a {@link ServletInputStream},
     * this method will be invoked by the container the first time when it is possible
     * to read data. Subsequently the container will invoke this method if and only
     * if {@link javax.servlet.ServletInputStream#isReady()} method
     * has been called and has returned <code>false</code>.
     *
      **/
    public void onDataAvailable() throws IOException;

    /**
     * Invoked when all data for the current request has been read.
     *
     * @throws IOException if an I/O related error has occurred during processing
     */    
    public void onAllDataRead() throws IOException;

    public void onError(Throwable t);

```
写监听器WriteListener
```java
public interface WriteListener extends EventListener {

    /**
     * When an instance of the WriteListener is registered with a {@link ServletOutputStream},
     * this method will be invoked by the container the first time when it is possible
     * to write data. Subsequently the container will invoke this method if and only
     * if {@link javax.servlet.ServletOutputStream#isReady()} method
     * has been called and has returned <code>false</code>.
     *
     * @throws IOException if an I/O related error has occurred during processing
     */
    public void onWritePossible() throws IOException;

    /**
     * Invoked when an error occurs writing data using the non-blocking APIs.
     */
    public void onError(final Throwable t);

}
```

异步上下文AsyncContext 

## RPC