# Tomcat总体结构图

<img src="https://tva1.sinaimg.cn/large/00831rSTgy1gdamkwyuvtj30yy0u0mzc.jpg" alt="截屏2020-03-29 上午11.13.11" style="zoom:50%;" />

# Tomcat启动流程

<img src="https://tva1.sinaimg.cn/large/00831rSTgy1gdamoahym8j310m0s8ac7.jpg" alt="截屏2020-03-29 上午11.15.58" style="zoom:50%;" />

# 容器的请求处理流程

<img src="https://tva1.sinaimg.cn/large/00831rSTgy1gdamqv4rqcj30u00wlgp4.jpg" alt="截屏2020-03-29 上午11.19.08" style="zoom:50%;" />

# Servlet容器工作原理

* 创建Request对象
* 创建Response对象
* 调用 servlet 的 service 方法，并传入 request 和 response 对象

Catalina是一个开源的servlet容器，它主要由两个模块组成：连接器和容器。连接器用来构造request和response对象，容器从连接器接收到 requset 和 response 对象之后调用 servlet 的 service 方法用于响应。

servlet 容器会为 servlet 的每个 HTTP 请求做下面一些工作:

* 当第一次调用 servlet 的时候，加载该 servlet 类并调用 servlet 的 init 方法
* 对每次请求，构造一个 javax.servlet.ServletRequest 实例和一个 javax.servlet.ServletResponse 实例。
* 调用 servlet 的 service 方法，同时传递 ServletRequest 和 ServletResponse 对象。
* 当 servlet 类被关闭的时候，调用 servlet 的 destroy 方法并卸载 servlet 类。

# http协议

http请求：

* 请求方法 请求地址 协议/版本
* 请求头部
* 请求内容

http响应：

* 协议/版本 状态码
* 响应头部
* 响应内容

# Tomcat默认连接器

一个 Tomcat 连接器必须符合以下条件：

* 必须实现接口 org.apache.catalina.Connector。
* 必须创建请求对象，该请求对象的类必须实现接口 org.apache.catalina.Request。
* 必须创建响应对象，该响应对象的类必须实现接口 org.apache.catalina.Response。

http1.1新特性：

* 持久连接
* 块编码
* 状态100

Connector和Container是一对一的关系。Connector知道Container 但反过来就不成立了。

## HttpConnector

HttpConnector是connector接口的一个实现类。

HttpConnector从一个服务端套接字工厂中获得一个 ServerSocket 实例。HttpConnector维护一个HttpProcessor连接池，可以同时处理多个http请求，连接池大小可以设置一个最大和最小值，开始的时候连接池中会创建最小值指定的实例数，超过最大连接数发过来的请求会被忽略。

# 容器

有四种类型的容器：

* Engine
* Host
* Context 表示一个web容器
* Wrapper 表示独立的servlet

## 流水线任务（pipeline）

一个 pipeline 包含了该容器要唤醒的所有任务。每一个valve表示了一个特定的 任务，一个容器的流水线有一个基本的valve。一个pipeline就像一个过滤链，每一个valve像一个过滤器。跟过滤器一样，一个valve可以操作传递给它的 request 和 response 方法。当一个valve完成了处理，则进一步处理pipeline中的下一个valve，基本valve总是在最后才被调用。

包含子容器的工作模型：

* 一个容器有一个pipeline，容器的 invoke 方法会调用pipeline的 invoke 方法。
* pipeline的 invoke 方法会调用添加到容器中的valve的 invoke 方法， 然后调用基本valve的 invoke 方法。
* 在一个wrapper中，基本valve负责加载相关的 servlet 类并对请求作出相应。
* 在一个有子容器的上下文中，基本valve使用 mapper 来查找负责处理请求的子容器。如果一个子容器被找到，子容器的 invoke 方法会被调用，然后返回步骤 1。

# 生命周期

实现了 Lifecycle 接口的组件同时会触发一个或多个下列事件：

* BEFORE_START_EVENT
* START_EVENT
* AFTER_START_EVENT
* BEFORE_STOP_EVENT
* STOP_EVENT
* AFTER_STOP_EVENT

当组件被启动的时候前三个事件会被触发，而组件停止的时候会触发后边三个事件。另外如果一个组件可以触发事件，那么必须存在相应的监听器来对触发的事件作出回应。

# 加载器

tomcat需要一个定制的加载器是因为如果使用系统加载器，那么servlet 就可以进入 Java 虚拟机 CLASSPATH 环境下面的任何类和类库带来安全隐患，servlet 只允许访问 WEB-INF/目录及其子目录下面的类以及部 署在WEB-INF/lib目录下的类库。此外tomcat需要支持在 WEB-INF/classes 或者是 WEB-INF/lib 目录被改变的时候会重新加载。

invoke 方 法被调用的时候，容器首先调用加载器的 getClassLoader 方法获得一个加载器，然后容器调用 loadClass 方法来加载 servlet 类。加载器会有单独的线程运行监测程序判断servlet类是否发生改变。

为了提高性能，当一个类被加载的时候会被放到缓存中，这样下次需要加载该类 的时候直接从缓存中调用即可。

# session管理

session对象通过一个叫管理器的的组建进行维护，默认情况下管理器将session对象存储在内存中，但是tomcat也允许将session对象存储在文件或者数据库中。

# StandardWrapper

一个servlet实现了SingleThreadModel接口表示这个servlet一次只能处理一个请求，也就是说保证不会有两个线程同时使用该servlet的service方法，但是该接口并不能保证线程安全。