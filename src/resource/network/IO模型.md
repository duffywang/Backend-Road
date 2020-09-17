## IO模型


### BIO
同步阻塞 1:1
在活动连接数不是特别高（小于单机1000）的情况下，这种模型是比较不错的，可以让每一个连接专注于自己的 I/O 并且编程模型简单，也不用过多考虑系统的过载、限流等问题。
线程池本身就是一个天然的漏斗，可以缓冲一些系统处理不了的连接或请求。但是，当面对十万甚至百万级连接的时候，传统的 BIO 模型是无能为力的。因此，我们需要一种更高效的 I/O 处理模型来应对更高的并发量。

### NIO
同步非阻塞 N:1
适用于高负载、高并发的（网络）应用
采取Reactor模式，或者说是Observer模式为我们监察I/O端口，其中将Channel注册到Selector上，指定感兴趣的具体事件，Selector类似于观察者，


### AIO
异步、非阻塞 M:0



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