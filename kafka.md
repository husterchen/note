# kafka相关

对于kafka来说存储的数据就是字节数组，不存在任何特殊的格式或者含义。

为了提高效率，消息被分批次写入Kafka。批次就是一组消息，这些消息属于同一个主题和分区。如果每一个消息都单独穿行于网络，会导致大量的网络开销，把消息分成批次传输可以减少网络开销。

由于一个主题一般包含几个分区，因此无法在整个主题范围内保证消息的顺序，但可以保证消息在单个分区内的顺序。Kafka通过分区来实现数据冗余和伸缩性。分区可以分布在不同的服务器上，也就是说，一个主题可以横跨多个服务器，以此来提供比单个服务器更强大的性能。

# broker 配置

常规配置

* broker.id
* port
* zookeeper.connect
* log.dirs
* num.recovery.threads.per.data.dir
* auto.create.topics.enable

主题默认配置：

* num.partitions
* log.retention.ms
* log.retention.bytes
* log.segment.bytes
* log.segment.ms
* message.max.bytes

# kafka生产者

## kafka生产组件：

![截屏2020-01-01下午4.34.57](https://tva1.sinaimg.cn/large/006tNbRwgy1gah5c2ogf9j30ui0u040o.jpg)

## 生产者配置：

* bootstrap.server
* key.serializer
* value.serializer
* acks //指定多少分区副本收到信息才会认为写入成功
* buffer.memory //生产者内存缓冲区大小
* compression.type
* retries
* batch.size
* linger.ms //发送批次前的等待时间
* client.id
* max.in.flight.requests.per.connection //生产者在收到服务器响应之前可以发送的消息数
* timeout.ms //同步副本超时时间
* request.timeout.ms //请求响应的超时时间
* metadata.fetch.timeout.ms //生产获取元数据的超时时间
* max.block.ms //允许阻塞时长
* max.request.size
* receive.buffer.bytes和send.buffer.bytes //针对于tcp连接

生产者消息发送方式：

* 发送并忘记
* 同步发送
* 异步发送（提供回调函数）

生产者一般会发生两类错误：可重试错误和非可重试错误，生产者可以配置为自动重试，对于可重试错误生产者或发起重试，多次重试无效返回异常，但是对于非可重试错误，比如“消息太大”异常，生产者不会进行重试直接返回异常。

## 序列化

* 自定义序列化方法：实现Serializer<?>接口
* 内置序列化器
* 第三方序列化器，例如：JSON、Avro、Thrift或Protobuf

## 分区

* 自定义分区：实现Partitioner接口
* 默认分区策略：散列

# kafka消费者

## 消费者群组

Kafka消费者从属于消费者群组。一个群组里的消费者订阅的是同一个主题，每个消费者接收主题一部分分区的消息。群组之间对主题分区不会进行瓜分，单个群组都能获取到主题的全部消息。

