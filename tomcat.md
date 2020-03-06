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



