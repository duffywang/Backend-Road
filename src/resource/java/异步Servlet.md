## 异步Servlet

### Servlet概述




### 异步Servlet
**Web应用程序中提供异步处理最基本的动机是去处理那些需要很长时间才能完成的请求。**
这些比较耗时的请求可能是一个缓慢的数据库查询，可能是对外部REST API的调用，也可能是其他一些耗时的I/O操作。
这种耗时较长的请求可能会快速耗尽容器线程池中的线程并影响应用的可伸缩性。
在Servlet3.0规范前，Servlet容器对Servlet都是以每个请求对应一个线程这种1:1的模式进行处理的，每当用户发起一个请求时，Tomcat容器就会分配一个线程来运行具体的Servlet。
在这种模式下，当在Servlet内执行比较耗时的操作，比如访问了数据库、同步调用了远程rpc，或者进行了比较耗时的计算时，当前分配给Servlet执行任务的线程会一直被该Servlet持有，不能及时释放掉供其他请求使用，而Tomcat内的容器线程池内线程是有限的，当线程池内线程用尽后就不能再对新来的请求进行及时处理了，所以这大大限制了服务器能提供的并发请求数量。为了解决这个问题，Servlet 3.0中引入了异步处理请求的能力，处理线程可以及时返回容器并执行其他任务，一个典型的异步处理的事件流程如下：

1. 请求被容器接收，然后从容器中获取一个线程来执行，请求被流转查找具体的Servlet进行处理。

2. Servlet内使用“req.startAsync()；”开启异步处理，返回异步处理上下文AsyncContext对象，然后开启异步线程（可以是Tomcat容器中的其他线程，也可以是业务自己创建的线程）对请求进行处理，开启异步线程后，当前Servlet就返回了，分配给其执行的容器线程也就释放了，并且不对请求方产生响应结果。

3. 异步线程对请求处理完毕后，会通过持有的AsyncContext对象把结果写回请求方，写入完毕关闭响应流。

异步Servlet的具体处理请求响应的逻辑已经不再是Servlet调用线程来做了，Servlet内开启异步处理后会立刻释放容器线程，具体对请求进行处理与响应的是业务线程池中的线程。


同步Servlet代码
```java
/* 1.注解方式标识Servlet */
@WebServlet(urlPatterns = "/test")
public class MyServlet extends HttpServlet {
    @Override
    protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        System.out.println("---begin servlet----");
        try { 
            /* 2.执行业务逻辑 */
            //do something

            /* 3.设置响应结果 */
            resp.setContentType("test/html");
            resp.getWriter().print("xxx");
        } catch (Exception e) {
            System.out.println(e.getMessage());
        } finally {}
        
        /* 4.运行结束，即将释放容器线程 */
        System.out.println("---end servlet----");
    }
}
```

使用容器线程池实现异步Servlet
```java
/** 1.开启异步支持，其中asyncSupported为true代表要异步执行
 */
@WebServlet(urlPatterns = "/test", asyncSupported = true)
public class MyServlet extends HttpServlet {
    @Override
    protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {

        /* 2.开启异步，调用HttpServletRequest的startAsync()方法开启异步调用，该方法返回一个AsyncContext异步上下文，其中保存了与请求/响应相关的上下文信息 */
        final AsyncContext asyncContext = req.startAsync();

		/* 3.提交异步任务，调用AsyncContext的start()方法并传递一个任务，service方法执行完毕后会马上返回，当前Servlet就退出了，其调用线程（此时使用的是容器线程）也被释放，
        * 提交异步任务后，异步任务的执行还是由容器中的其他线程来具体执行的。
        * */
        asyncContext.start(new Runnable() {
            @Override
            public void run() {
                try {
                    /* 3.1执行业务逻辑 */
                    //Do Something

                    /* 3.2设置响应结果，从asyncContext中获取响应对象，并把响应结果写入响应对象。*/
                    resp.setContentType("test/html");
                    PrintWriter out = asyncContext.getResponse().getWriter().println("xxxxx");
                } catch (Exception e) {
                    System.out.println(e.getLocalizedMessage());
                } finally {
                    /* 3.3异步完成通知，标识异步任务执行完毕 */
                    asyncContext.complete();
                }
            }
        });

        /* 4.运行结束，即将释放容器线程 */
        System.out.println("---end servlet----");
    }
}
```

使用自定义线程池实现异步Servlet
```java
/* 1.开启异步支持 */
@WebServlet(urlPatterns = "/test", asyncSupported = true)
public class MyServlet extends HttpServlet {
    /* 2.自定义线程池，与容器线程池的区别*/
    private final static int AVALIABLE_PROCESSORS = Runtime.getRuntime().availableProcessors();
    private final static ThreadPoolExecutor POOL_EXECUTOR = new ThreadPoolExecutor(AVALIABLE_PROCESSORS,
            AVALIABLE_PROCESSORS * 2, 1, TimeUnit.MINUTES, new LinkedBlockingQueue<>(5),
            new ThreadPoolExecutor.CallerRunsPolicy());

    @Override
    protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        /* 3.开启异步，获取异步上下文 */
        System.out.println("---begin servlet----");
        final AsyncContext asyncContext = req.startAsync();

        /* 4.提交异步任务 与容器线程池对比 */
        POOL_EXECUTOR.execute(new Runnable() {
            @Override
            public void run() {
                try {
                    /* 4.1执行业务逻辑 */
                    //Do Something

                    /* 4.2设置响应结果 */
                    resp.setContentType("test/html");
                    PrintWriter out = asyncContext.getResponse().getWriter().print("xxxx");
                } catch (Exception e) {
                    System.out.println(e.getLocalizedMessage());
                } finally {
                    /* 4.3异步完成通知 */
                    asyncContext.complete();
                }
            }
        });

        /* 5.运行结束，即将释放容器线程 */
        System.out.println("---end servlet----");
    }
}
```
通过如上方式，我们创建了JVM内全局的线程池，然后把异步任务提交到了我们的线程池来执行，这时候整个处理流程是：
Tomcat容器收到请求后，从容器中获取一个线程来执行容器层的一些处理流程，接着把请求同步转发到MyServlet的service方法来执行，然后使用req.startAsync()把具体请求处理的逻辑异步切换到我们业务线程池来执行，此时MyServlet就返回了，并释放容器线程。

### 初识AsyncContext
AsyncContext是一个接口

```java
public interface AsyncContext {
    /**
    * the request that was used to initialize this AsyncContext
    * */
    public ServletRequest getRequest();
    /**
    * the response that was used to initialize this AsyncContex
    * */
    public ServletResponse getResponse();
    /**
    * Causes the container to dispatch a thread, possibly from a managed thread pool, to run the specified Runnable
    * */
    public void start(Runnable run);
    /**
    * */        
    public void complete();
    /**
    * <p>The given AsyncListener will receive an AsyncEvent when
    * the asynchronous cycle completes successfully, times out, results
    * in an error, or a new asynchronous cycle is being initiated via
    * one of the  ServletRequest#startAsync methods.
    * */
    public void addListener(AsyncListener listener);
     /**
     * */       
    public <T extends AsyncListener> T createListener(Class<T> clazz)
                throws ServletException; 
    /**
    * <p>The timeout will expire if neither the {@link #complete} method
    * nor any of the dispatch methods are called. A timeout value of
    * zero or less indicates no timeout. 
    * The default value is <code>30000</code> ms.
    * */        
    public void setTimeout(long timeout);
    
    /**
     * Dispatches the request and response objects of this AsyncContext
     * to the given <tt>path</tt> scoped to the given <tt>context</tt>. 
    */
    public void dispatch();
        
        

}


```

AsyncListener

```java
public interface AsyncListener extends EventListener {
    
    public void onComplete(AsyncEvent event) throws IOException;
   
    public void onTimeout(AsyncEvent event) throws IOException;
    
    public void onError(AsyncEvent event) throws IOException;
    
    public void onStartAsync(AsyncEvent event) throws IOException;     


}
```

ReadListener
```java
public interface ReadListener extends EventListener {

    /**
     * When an instance of the <code>ReadListener</code> is registered with a ServletInputStream,
     * this method will be invoked by the container the first time when it is possible
     * to read data. Subsequently the container will invoke this method if and only
     * if  javax.servlet.ServletInputStream#isReady() method
     * has been called and has returned <code>false</code>.
     *
     */
    public void onDataAvailable() throws IOException;

    /**
     * Invoked when all data for the current request has been read.
     *
     */

    public void onAllDataRead() throws IOException;

    /**
     * Invoked when an error occurs processing the request.
     */
    public void onError(Throwable t);


}
```

WriteListener
```java
public interface WriteListener extends EventListener {

    /**
     * When an instance of the WriteListener is registered with a ServletOutputStream,
     * this method will be invoked by the container the first time when it is possible
     * to write data. Subsequently the container will invoke this method if and only
     * if javax.servlet.ServletOutputStream#isReady() method
     * has been called and has returned <code>false</code>.
     *
     */
    public void onWritePossible() throws IOException;

    /**
     * Invoked when an error occurs writing data using the non-blocking APIs.
     */
    public void onError(final Throwable t);

}
```

请求上下文RequestContext
```java

public class RequestContext implements Serializable{
    
    
    //Async Servlet
    private AsyncContext asyncContext;
    
    private AsyncListener asyncListener;
    
    private transient HttpServletRequest request;
    
    private transient HttpServletResponse response;
}
```