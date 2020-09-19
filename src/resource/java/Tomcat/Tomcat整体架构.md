# Tomcat

## 架构组件
### 顶级组件
1.Server：表示一个Tomcat实例，包含一个或多个Service子容器
2.Service：代表Tomcat中一组提供服务、处理请求的组件，包含多个Connector和Container
### 连接器
Connector是客户端连接到Tomcat容器的服务点，需要监听IP地址、端口号以及所支持的协议（HTTP、HTTPS、AJP、SSL、proxy）
IO模式（BIO、NIO、AIO、APR）,以及处理线程池参数（）

Connector结构

- Endpoint:实现TCP/IP协议，用来处理底层Socket的网络连接,
- Processor:用于将Endpoint接收到的Socket封装成Request，
- Adapter:用于将Request交给Container 中servlet进行具体的处理
### 容器组件
Container用于封装和管理Servlet，以及具体处理Request请求，设计使用责任链的设计模式，分别由四个容器构成：
- Engine ：表示可运行Catalina的servlet的引擎实例，包含了servlet 容器的核心内容
- Host ：作用是运行多个应用，负责安装和展开这些应用，一个虚拟主机下都可以部署一个或者多个Web App，Host主要是用解析web.xml
- Context：代表Servlet的上下文，具备Servlet运行的基本环境，管理其中的Servlet实例，一个Context代表一个Web Application，而一个Web Application 由一个或多个Servlet实例构成。理论上只要有 Context 就能运行 Servlet 了。简单的 Tomcat 可以没有 Engine 和 Host
- Wrapper: 代表一个Servlet，负责管理一个Servlet生命周期，包括加载、初始化、执行及资源回收

### 嵌套组件
- Valve:类似Servlet中定义的过滤器，用来拦截请求并在将其转至目标之前进行某种处理操作，可以定义在任何容器类的组件中，常用来记录客户端请求、客户端IP地址和服务器等信息，一般称为请求转储（请求转储valve记录请求客户端请求数据包中HTTP Header信息和Cookie信息）
- Logger:记录组件内部的状态信息
- Loader:负责加载、解释Java类编译后的字节码
- Realm:用于用户的认证和授权，管理员可以为每个资源定义权限
- Excutor:用于配置共享线程池，用于Connector使用
- Listener:监听已注册组件的生命周期
- Manager:用户管理HTTP会话
- Cluster:专用于配置Tomcat集群的元素，可用于Engine和Host容器
