# Dubbo 分层架构

* Service层
* Config层
* Proxy服务代理层
* Register服务注册中心层
* Cluster路由层
* Monitor监控层
* Protocol远程调用层
* Exchange信息交换层
* Transport网络传输层
* Serialize数据序列化层

# SPI机制

动态的为接口提供具体的实现，通过配置文件的形式制定接口的具体实现类并进行动态加载。

java中可以通过ServiceLoader定位获取给定接口的具体实现。

jdk的标准SPI会同时把拓展接口的所有实现类提前加载好实例。

# Dubbo 适配器模式

dubbo通过适配器模式实现拓展接口的功能。dubbo为每个拓展功能对应的接口通过动态编译技术生成一个适配器类，该适配器类根据不同的传参获取对应的拓展实现。整个过程先生成源码，然后进行动态编译和实例化。

# 服务暴露过程



<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1gicmbbvm1pj31ef0u07li.jpg" alt="截屏2020-09-02 下午9.17.22" style="zoom:50%;" />

dubbo的zk路径结构：

* consumers
* configurators 服务降级策略
* routers 路由信息
* providers

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1gict7r86otj31go0qido2.jpg" alt="截屏2020-09-03 上午1.16.09" style="zoom:50%;" />

# 服务消费过程

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1gicmc0mx8tj31fi0r8tqt.jpg" alt="截屏2020-09-02 下午9.18.04" style="zoom:50%;" />

默认情况当消费者引用同一个服务提供者机器上的多个服务时复用同一个Netty连接。

服务消费者和提供者的所有机器都有连接。

可以配置惰性连接，默认为false，在消费者启动时就与提供者建立了连接。

# Directory目录和Router路由服务

Directory实现：

* RegistryDirectory：invoker列表是根据服务注册中心的推送变化而变化。
* StaticDirectory：当消费端使用了多注册中心时，其把所有服务注册中心的invoker列表汇集到一个invoker列表中。

# 服务降级

服务降级策略是在远程调用前应用，根据url是否包含降级信息来选择具体的调用策略。

服务mock和服务降级具体实现类似，但是前者会动态创建mock类。

# 集群容错和负载均衡

Dubbo提供的集群容错模式：

* Failover Cluster：失败重试，自动切换到其他服务提供者机器进行重试，每次重试会重新更新服务列表信息。
* Failfast Cluster：快速失败，调用失败立即报错
* Failsafe Cluster：安全失败，调用失败忽略异常
* Failback Cluster：失败自动恢复，在后台记录失败请求，按照一定的策略后期进行重试
* Forking Cluster：并行调用，并行调用多个服务提供者，只要其中有一个成功即返回
* Broadcast Cluster：广播调用，逐个调用所有的服务提供者，只要有一台服务器调用失败就失败

除了上面提到的Dubbo内置的容错机制，我们还可以通过实现Cluster接口编写自己的容错策略。

Dubbo提供的负载均衡策略：

* Random LoadBalance 权重
* RoundRobin LoadBalance 权重
* LeastActive LoadBalance
* ConsistentHashLoadBalance

一般的hash算法当新增或服务器宕机时会存在比较大的波动，如果存在缓存会导致大量的缓存失效，也无法根据机器的性能区别利用。

可以通过实现LoadBalance定制化负载策略。

# Dubbo线程模型和线程池策略

Dubbo默认的底层网络通信使用的是Netty，服务提供方NettyServer使用两级线程池，其中EventLoopGroup（boss）主要用来接收客户端的链接请求，并把完成TCP三次握手的连接分发给EventLoopGroup（worker）来处理，我们把boss和worker线程组称为I/O线程。

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1gih8afbzg2j31vg0cegs7.jpg" alt="截屏2020-09-06 下午8.59.56" style="zoom:50%;" />

Dubbo提供了以下几种线程模型：

* all：所有消息派发到业务线程池
* direct：所有消息全部在IO线程上直接执行
* message：只有请求响应消息派发到业务线程池
* execution：只把请求类消息派发到业务线程池处理
* connection：在I/O线程上将连接事件，断开事件放入队列有序逐个执行，其他消息派发到业务线程池

Dubbo线程池实现：

* FixedThreadPool：具有固定大小的线程池。（默认）
* LimitedThreadPool：线程池中的线程个数随着需要量动态增加，但是数量不超过配置的阈值，空闲线程不会被回收。
* EagerThreadPool：当线程池中所有核心线程都处于忙碌状态时，将创建新的线程来执行新任务。
* CachedThreadPool：当线程空闲1分钟时，线程会被回收；当有新请求到来时，会创建新线程。

# 泛化引用

泛化接口调用方式主要是在服务消费端没有API接口类及模型类元（比如入参和出参的POJO类）的情况下使用，其参数及返回值中没有对应的POJO类，所以全部POJO均转换为Map表示。使用泛化调用时服务消费模块不再需要引入SDK二方包。

服务消费端使用GenericImplFilter拦截泛化调用，把泛化参数进行校验并发起远程调用；服务提供方使用GenericFilter拦截请求，并把泛化参数进行反序列化处理，然后把请求转发给具体的服务进行执行。

# Dubbo并发控制

consumer：ActiveLimitFilter

Provider：ExecuteLimitFilter

# Dubbo 隐士参数传递

服务调用方可以通过RpcContext.getContext（）.setAttachment（）方法设置附加属性键值对，然后设置的键值对可以在服务提供方服务方法内获取。

要实现隐式参数传递，首先需要在服务消费端的AbstractClusterInvoker类的invoke（）方法内，把附加属性键值对放入到RpcInvocation的attachments变量中，然后经过网络传递到服务提供端；服务提供端则使用ContextFilter对请求进行拦截，并从RpcInvocation中获取attachments中的键值对，然后使用RpcContext.getContext（）.setAttachment设置到上下文对象中。

# Dubbo全链路异步

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1gihaf98l5hj31z40to1ky.jpg" alt="截屏2020-09-06 下午10.13.47" style="zoom:50%;" />

Dubbo以CompletableFuture为基础支持所有异步编程接口。老版本的异步实现需要用户线程调用future对象的get，该方法的调用会导致用户线程阻塞，当然可以通过为future对象设置回调。

服务提供端的异步执行可以将服务的处理逻辑从Dubbo的内部线程池切换到业务自定义线程。

# dubbo应用

## 启动时检查

```xml
<dubbo:reference interface="com.foo.BarService" check="false" />
<dubbo:consumer check="false" />
<dubbo:registry check="false" />
```

## 集群容错

```xml
<dubbo:service retries="2" />
<dubbo:service cluster="failsafe" />
```

## 负载均衡

```xml
<dubbo:service interface="..." loadbalance="roundrobin" />
<dubbo:service interface="...">
    <dubbo:method name="..." loadbalance="roundrobin"/>
</dubbo:service>
```

## 线程模型

```xml
线程池选型：fixed，cached，limited，eager
<dubbo:protocol name="dubbo" dispatcher="all" threadpool="fixed" threads="100" />
```

## 点对点直连

```xml
<dubbo:reference id="xxxService" interface="com.alibaba.xxx.XxxService" url="dubbo://localhost:20890" />
```

## 只订阅/只注册

```xml
<dubbo:registry address="10.20.153.10:9090" register="false" />
```

## 静态服务

```xml
服务启动不会注册，服务关闭也不会自动删除
<dubbo:registry address="10.20.141.150:9090" dynamic="false" />
```

## 多协议/多注册中心

```xml
多协议
<dubbo:protocol name="dubbo" port="20880" />
<dubbo:service interface="" version="1.0.0" ref="helloService" protocol="dubbo" />
多注册中心
<dubbo:registry id="hangzhouRegistry" address="10.20.141.150:9090" />
<dubbo:service interface="" version="1.0.0" ref="helloService" registry="hangzhouRegistry" />
```

## 服务分组/多版本

```xml
<dubbo:service group="feedback" interface="com.xxx.IndexService" />
<dubbo:reference id="feedbackIndexService" group="feedback" interface="com.xxx.IndexService" />
```

## 分组聚合

```xml
<dubbo:reference interface="com.xxx.MenuService" group="*" merger="true" />
```

## 参数验证

```xml
<dubbo:reference id="validationService" interface="ValidationService" validation="true" />
```

## 结果缓存

```xml
<dubbo:reference interface="com.foo.BarService" cache="lru" />
```

## 泛化调用/泛化实现

```xml
<dubbo:reference id="barService" interface="com.foo.BarService" generic="true" />
泛化实现通过实现GenericService接口进行服务编写
```

## 回声测试

所有服务自动实现EchoService，引用时强转。

## 上下文信息RpcContext

也可以用来隐式传递参数

```java
RpcContext.getContext().setAttachment("index", "1");
RpcContext.getContext().getAttachment("index");
```

## 异步调用

需要服务提供者定义CompletableFuture类型的返回值。

可以通过调用服务获取CompletableFuture，也可以从RpcContext中获取。

## 异步实现

* CompletableFuture
* AsyncContext

## 本地调用

协议设置成injvm，从 `2.2.0` 开始，每个服务默认都会在本地暴露。在引用服务的时候，默认优先引用本地服务。

## 参数回调

服务提供者需要配置回调类型的参数

```xml
<bean id="callbackService" class="com.callback.impl.CallbackServiceImpl" />
<dubbo:service interface="com.callback.CallbackService" ref="callbackService" connections="1" callbacks="1000">
    <dubbo:method name="addListener">
        <dubbo:argument index="1" callback="true" />
        <!--也可以通过指定类型的方式-->
        <!--<dubbo:argument type="com.demo.CallbackListener" callback="true" />-->
    </dubbo:method>
</dubbo:service>
```

## 事件通知

在调用之前、调用之后、出现异常时，会触发 `oninvoke`、`onreturn`、`onthrow` 三个事件.

```xml
<bean id ="demoCallback" class = "NofifyImpl" />
<dubbo:reference id="demoService" interface="IDemoService" version="1.0.0" group="cn" >
      <dubbo:method name="get" async="true" onreturn = "demoCallback.onreturn" onthrow="demoCallback.onthrow" />
</dubbo:reference>
```

## 本地存根

相当于客户端代理，需要在客户端配置stub代理服务。

```xml
<dubbo:service interface="com.foo.BarService" stub="com.foo.BarServiceStub" />
```

## 本地伪装

就是服务降级mock。

除了自定义mock行为，dubbo提供了以下几种简单的mock配置：

* return（empty，null，true，false，JSON）
* throw
* force 强制mock，不走远程
* fail 调用失败强制mock

```xml
<dubbo:reference interface="com.foo.BarService" mock="com.foo.BarServiceMock" />
<dubbo:reference id="demoService" check="false" interface="com.foo.BarService">
    <dubbo:parameter key="sayHello.mock" value="force:return fake"/>
</dubbo:reference>
```

## 服务延迟暴露

计时从Spring初始化完成后开始进行。

```xml
<dubbo:service delay="-1" />
```

## 并发/连接控制

```xml
// 服务端
<dubbo:service interface="com.foo.BarService" executes="10" />
<dubbo:provider protocol="dubbo" accepts="10" />
// 客户端
<dubbo:service interface="com.foo.BarService" actives="10" />
<dubbo:reference interface="com.foo.BarService" connections="10" />
```

## 延迟连接

```xml
// 该配置只对使用长连接的 dubbo 协议生效
<dubbo:protocol name="dubbo" lazy="true" />
```

## 粘滞连接

粘滞连接用于有状态服务，尽可能让客户端总是向同一提供者发起调用，除非该提供者挂了，再连另一台。粘滞连接将自动开启延迟连接，以减少长连接数。

```xml
<dubbo:reference id="xxxService" interface="com.xxx.XxxService" sticky="true" />
```

## 令牌验证

通过令牌验证在注册中心控制权限，以决定要不要下发令牌给消费者，可以防止消费者绕过注册中心访问提供者。

```xml
<dubbo:provider interface="com.foo.BarService" token="true" />
<dubbo:service interface="com.foo.BarService" token="true" />
```

