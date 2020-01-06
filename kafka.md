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

## 分区再均衡

再均衡期间消费者无法读取消息，造成群组一段时间内的不可用。另外，当分区被重新分配给另一个消费者时，消费者当前的读取状态会丢失，它有可能还需要去刷新缓存，在它重新恢复状态之前会拖慢应用程序。

消费者通过向被指派为群组协调器的broker（不同的群组可以有不同的协调器）发送心跳来维持它们和群组的从属关系以及它们对分区的所有权关系。

当消费者要加入群组时，它会向群组协调器发送一个JoinGroup请求。第一个加入群组的消费者将成为“群主”。群主从协调器那里获得群组的成员列表（列表中包含了所有最近发送过心跳的消费者，它们被认为是活跃的），并负责给每一个消费者分配分区。分配完毕之后，群主把分配情况列表发送给群组协调器，协调器再把这些信息发送给所有消费者。每个消费者只能看到自己的分配信息，只有群主知道群组里所有消费者的分区信息。

## 消费者配置

* bootstrap.servers
* group.id
* key.deserializer
* value.deserializer
* fetch.min.bytes //收到请求时会等到有足够的可用数据时才把它返回给消费者
* fetch.max.wait.ms
* max.partition.fetch.bytes
* session.timeout.ms //超时死亡时间
* auto.offset.reset 指定了消费者读取一个没有偏移量的分区或者偏移量无效的情况下如何处理
* enable.auto.commit //是否自动提交偏移量
* partition.assignment.strategy //分区策略：Range和RoundRobin
* client.id
* max.poll.records
* receive.buffer.bytes和send.buffer.bytes

## 偏移量

老版本是通过zookeeper保存偏移量的。

消费者往一个叫作_consumer_offset的特殊主题发送消息，消息里包含每个分区的偏移量。如果消费者发生崩溃或者有新的消费者加入群组，就会触发再均衡，完成再均衡之后，每个消费者可能分配到新的分区，而不是之前处理的那个。为了能够继续之前的工作，消费者需要读取每个分区最后一次提交的偏移量，然后从偏移量指定的地方继续处理。

偏移量提交方式：

* 自动提交
* commitSync()
* commitAsync()

## 再均衡监听

在为消费者分配新分区或移除旧分区时，可以通过消费者API执行一些应用程序代码，在调用subscribe()方法时传进去一个ConsumerRebalanceListener实例就可以了。ConsumerRebalanceListener有两个需要实现的方法：

* public void onPartitionsRevoked(Collection<TopicPartition> partitions)方法会在再均衡开始之前和消费者停止读取消息之后被调用。如果在这里提交偏移量，下一个接管分区的消费者就知道该从哪里开始读取了。
* public void onPartitionsAssigned(Collection<TopicPartition> partitions)方法会在重新分配分区之后和消费者开始读取消息之前被调用。

## 从特定分区获取数据

如果你想从分区的起始位置开始读取消息，或者直接跳到分区的末尾开始读取消息，可以使用seekToBeginning(Collection<TopicPartition> tp)和seekToEnd(Collection<TopicPartition> tp)这两个方法。此外还提供了seek方法可以定位到指定分区，seek方法的 定位只是暂时的，下次调用poll时还是会获取最新的未消费数据。

## 消费者退出循环

如果确定要退出循环，需要通过另一个线程调用consumer.wakeup()方法。如果循环运行在主线程里，可以在ShutdownHook里调用该方法。要记住，consumer.wakeup()是消费者唯一一个可以从其他线程里安全调用的方法。调用consumer.wakeup()可以退出poll()，并抛出WakeupException异常，或者如果调用consumer.wakeup()时线程没有等待轮询，那么异常将在下一轮调用poll()时抛出。我们不需要处理WakeupException，因为它只是用于跳出循环的一种方式。不过，在退出线程之前调用consumer.close()是很有必要的，它会提交任何还没有提交的东西，并向群组协调器发送消息，告知自己要离开群组，接下来就会触发再均衡，而不需要等待会话超时。

## 独立消费者

独立消费者可以通过指定消费分区，该情况不需要考虑分区再均衡的情况，当然当添加了新的分区该消费者也无法得知，具体调用方法：consumer.assign()。

# kafka内部工作原理

## 控制器

负责分区首领的选举，集群里第一个启动的broker通过在Zookeeper里创建一个临时节点/controller让自己成为控制器。其他broker在启动时也会尝试创建这个节点，不过它们会收到一个“节点已存在”的异常，然后“意识”到控制器节点已存在，也就是说集群里已经有一个控制器了。其他broker在控制器节点上创建Zookeeperwatch对象，这样它们就可以收到这个节点的变更通知。

## 复制

分区分本分为首领副本和跟随者副本，跟随者副本不处理请求，只是复制首领分区消息。首领分区还会记录哪个跟随者是和自己一致的，跟随者副本会定时向首领副本发送请求信息，如果指定时间内没有收到某一跟随者对于最新消息的请求那么会认为该跟随者副本不同步，不同步的跟随者副本不能被选为新的首领。

## 请求处理

![截屏2020-01-04下午7.13.19](https://tva1.sinaimg.cn/large/006tNbRwgy1gakqsdtcmfj31d60rq45a.jpg)

broker会在它所监听的每一个端口上运行一个Acceptor线程，这个线程会创建一个连接，并把它交给Processor线程去处理。Processor线程（也被叫作“网络线程”）的数量是可配置的。网络线程负责从客户端获取请求消息，把它们放进请求队列，然后从响应队列获取响应消息，把它们发送给客户端。![截屏2020-01-04下午7.22.12](https://tva1.sinaimg.cn/large/006tNbRwgy1gakr0zwj1aj317q0u0why.jpg)

客户端发送请求前会发送元数据获取请求获得感兴趣主题所在的borker，客户端会缓存该元数据信息，每隔一段时间会发送一次元数据请求跟新客户端缓存，另外如果客户端收到非首领错误也会重新获取元数据。

在处理获取请求时Kafka使用零复制技术向客户端发送消息——也就是说，Kafka直接把消息从文件（或者更确切地说是Linux文件系统缓存）里发送到网络通道，而不需要经过任何中间缓冲区。

分区首领知道每个消息会被复制到哪个副本上，在消息还没有被写入所有同步副本之前，是不会发送给消费者的——尝试获取这些消息的请求会得到空的响应而不是错误。

## 物理存储

Kafka的基本存储单元是分区。创建主题后kafka会在broker之间分配分区，分配完之后为分区分配文件目录，计算每个目录里的分区数量，新的分区总是被添加到数量最小的那个目录里。

因为在一个大文件里查找和删除消息是很费时的，也很容易出错，所以kafka把分区分成若干个片段。默认情况下，每个片段包含1GB或一周的数据，以较小的那个为准。在broker往分区写入数据时，如果达到片段上限，就关闭当前文件，并打开一个新文件。当前正在写入数据的片段叫作活跃片段，活动片段永远不会被删除。

如果生产者发送的是压缩过的消息，那么同一个批次的消息会被压缩在一起，被当作“包装消息”进行发送。于是broker就会收到一个这样的消息，然后再把它发送给消费者。消费者在解压这个消息之后，会看到整个批次的消息，它们都有自己的时间戳和偏移量。

为了帮助broker更快地定位到指定的偏移量，Kafka为每个分区维护了一个索引。索引把偏移量映射到片段文件和偏移量在文件里的位置。

## compact策略

如果在Kafka启动时启用了清理功能（通过配置log.cleaner.enabled参数），每个broker会启动一个清理管理器线程和多个清理线程，它们负责执行清理任务。这些线程会选择污浊率（污浊消息占分区总大小的比例）较高的分区进行清理。清理任务只保留同一键值对应的最新消息。在compact策略下删除一条消息时，首先会将该消息的值置空并保留一段时间，时间过了才删除。

# kafka可靠性

## 相关配置

kafka系统配置：

* 复制系数。主题级别的配置参数是replication.factor，而在broker级别则可以通过default.replication.factor来配置自动创建的主题。
* unclean.leader.election。是否允许不同步的分区副本被选举为首领，只能在broker级别进行配置。
* min.insync.replicas。最少同步副本数，表示数据最少被写入多少副本才算已提交。


生产者配置：

* 重试次数
* 发送确认
* 恰当的错误处理

消费者配置：（原则就是可靠的偏移量的提交）

* auto.offset.reset。这个参数指定了在没有偏移量可提交时或者请求的偏移量在broker上不存在时消费者会做些什么。earliest表示从分区的最开始读取，latest表示从分区的末尾开始读取。
* enable.auto.commit
* auto.commit.interval.ms

Kafka提供了两个重要的工具用于验证配置：VerifiableProducer和VerifiableConsumer这两个类。

# 构建数据管道

## Kafka Connect

# MirrorMaker

# Kafka管理工具

* kafka-topics.sh 主题管理工具

```shell
//创建主题
kafka topics.sh --zookeeper <zookeeperconnect> --create --topic <string> --replication-factor <integer> --partitions <integer>
//修改主题，不能减少
--alter
--if-not-exists
--delete
--list //列出集群中所有主题
--describe （--topic） //列出所有主题的详细信息
--describe --under-replicated-partitions //列出所有包含不同步副本的分区
--describe --unavailable-partitions //列出所有没有首领的分区
```

* kafka-consumer-groups.sh 消费者群组工具
* kafka-run-class.sh kafka.tools.ExportZkOffsets 导出消费者群组的偏移量
* kafka-run-class.sh kafka.tools.ImportZkOffsets 导入消费者群组的偏移量
* kafka-configs.sh 动态配置变更
* kafka-preferred-replica-election.sh 手动触发首领副本选举
* kafka-reassign-partitions.sh 分配分区副本
* kafka-run-class.sh kafka.tools.DumpLogSegments 转储日志片段
* kafka-replica-verification.sh 验证分区副本的一致性
* kafka-console-consumer.sh
* kafka-console-producer.sh
