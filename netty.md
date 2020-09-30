# Netty体系结构

主旨：Netty是异步和事件驱动的

## Netty主要组件

* Channel

  看一看做读写数据的载体。

* 回调

* Future

  Netty实现了ChannelFuture，可以通过注册回调函数获取结果而不是阻塞调用get方法。

* 事件和ChannelHandler

每个Channel都拥有一个与之相关联的ChannelPipeline，其持有一个ChannelHandler的实例链。当一个新的连接被接受时，一个新的子Channel将会被创建，而ChannelInitializer将会把Handler的实例添加到该Channel的ChannelPipeline中。

Netty基础组件之间的关系图：

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1gix5gqynadj312w0u0b29.jpg" alt="截屏2020-09-20 下午3.31.11" style="zoom:50%;" />

## Netty提供的网络传输方式

* NIO
* Epoll
* OIO
* Local
* Embedded

## ByteBuf

Netty的数据容器，和java提供的ByteBuffer相比存在以下优势：

* 它可以被用户自定义的缓冲区类型扩展；
* 通过内置的复合缓冲区类型实现了透明的零拷贝；
* 容量可以按需增长（类似于JDK的StringBuilder）；
* 在读和写这两种模式之间切换不需要调用ByteBuffer的flip()方法；
* 读和写使用了不同的索引；
* 支持方法的链式调用；
* 支持引用计数；
* 支持池化。

ByteBuf引入了引用计数技术。

## ByteBufAllocator

ByteBuf池化。

## ChannelHandler

channel生命周期：

* ChannelUnregistered //Channel已经被创建，但还未注册到EventLoop
* ChannelRegistered //Channel已经被注册到了EventLoop
* ChannelActive //Channel处于活动状态（已经连接到它的远程节点），它现在可以接收和发送数据了
* ChannelInactive //Channel没有连接到远程节点

当这些状态发生改变时会生成对应的事件。这些事件将会被转发给ChannelPipeline中的ChannelHandler，其可以随后对它们做出响应。

channelHandler生命周期：

* handlerAdded //当把ChannelHandler添加到ChannelPipeline中时被调用
* handlerRemoved //当从ChannelPipeline中移除ChannelHandler时被调用
* exceptionCaught //当处理过程中在ChannelPipeline中有错误产生时被调用

## EventLoop

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1giynjm9ricj31ru0u0x6p.jpg" alt="截屏2020-09-21 下午10.42.20" style="zoom:50%;" />

不要将一个长时间运行的任务放入到执行队列中，因为它将阻塞需要在同一线程上执行的任何其他任务。

异步传输：

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1giynqbypzuj31ii0u0npe.jpg" alt="截屏2020-09-21 下午10.48.50" style="zoom:50%;" />

阻塞传输：

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1giynr4w5cjj31tl0u0u0x.jpg" alt="截屏2020-09-21 下午10.49.36" style="zoom:50%;" />

Netty的模型是基于Reactor模式。Netty的NioEventLoop模型和dubbo的网络模型差不多分为Boss Group和Worker Group。

[线程模型的具体讲解](https://ifeve.com/%E8%B0%88%E8%B0%88netty%E7%9A%84%E7%BA%BF%E7%A8%8B%E6%A8%A1%E5%9E%8B/)

## 客户端引导Bootstrap

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1gj1qxsh0buj31ht0u0kjl.jpg" alt="截屏2020-09-24 下午2.56.27" style="zoom:50%;" />

## 服务端引导ServerBootstrap

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1gj1qyw5tjnj326n0u04qp.jpg" alt="截屏2020-09-24 下午2.57.33" style="zoom:50%;" />

从channel引导客户端：

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1gj1qzd1xehj31lz0u07wi.jpg" alt="截屏2020-09-24 下午2.58.01" style="zoom:50%;" />

# Netty网络知识

通过netty编码和解码组件可以解决tcp粘包和拆包。