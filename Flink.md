# 基础知识

数据流上的操作：

* 输入输出
* 转换操作
* 滚动聚合
* 窗口操作

窗口类型：

* 滚动窗口
* 滑动窗口
* 会话窗口

流式应用的时间概念：

* 处理时间
* 事件时间

流处理的结果保证：

* 至多一次
* 至少一次
* 精确一次
* 端到端精确一次

# Flink架构

主要组件：

* Dispatcher
* JobManager
* ResourceManager
* TaskManager

TaskManager会在同一个JVM中以多线程的方式执行任务，但是这样会导致只要有一个任务运行异常就会导致整个TaskManager进程被杀死。

# Flink高可用

