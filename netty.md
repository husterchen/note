# Netty体系结构

主旨：Netty是异步和事件驱动的

## Netty主要组件

* Channel

  看一看做读写数据的载体。

* 回调

* Future

  Netty实现了ChannelFuture，可以通过注册回调函数获取结果而不是阻塞调用get方法。

* 事件和ChannelHandler