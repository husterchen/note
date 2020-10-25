# kafka相关

对于kafka来说存储的数据就是字节数组，不存在任何特殊的格式或者含义。

为了提高效率，消息被分批次写入Kafka。批次就是一组消息，这些消息属于同一个主题和分区。如果每一个消息都单独穿行于网络，会导致大量的网络开销，把消息分成批次传输可以减少网络开销。

由于一个主题一般包含几个分区，因此无法在整个主题范围内保证消息的顺序，但可以保证消息在单个分区内的顺序。Kafka通过分区来实现数据冗余和伸缩性。分区可以分布在不同的服务器上，也就是说，一个主题可以横跨多个服务器，以此来提供比单个服务器更强大的性能。

kafka持久化采用消息日志的形式，日志只能追加写，避免了随机I/O。在 Kafka 底层，一个日志又近一步细分成多个日志段，消息被追加写到当前最新的日志段中，当写满了一个日志段后，Kafka 会自动切分出一个新的日志段，并将老的日志段封存起来。Kafka 在后台还有定时任务会定期地检查老的日志段是否能够被删除，从而实现回收磁盘空间的目的。

在 Kafka 中实现这种 P2P 模型的方法就是引入了消费者组（Consumer Group）。所谓的消费者组，指的是多个消费者实例共同组成一个组来消费一组主题。这组主题中的每个分区都只会被组内的一个消费者实例消费，其他消费者实例不能消费它。

实际上 Kafka 客户端底层使用了 Java 的 selector，selector 在 Linux 上的实现机制是 epoll，而在 Windows 平台上的实现机制是 select。

kafka的消息发送属于异步调用，对于消息是否发送成功我们并不知道，所以尽量使用带有回调函数的发送api。

kafka消费者有时会出现消费的数据丢失，可能是因为更新了消费的偏移量但并没有对数据进行消费。

当消费者利用多线程进行消息处理时，如果开启了自动提交偏移量，当一个消息处理线程发生错误，但此时消息的偏移量已经更新，那么该条消息就不会被处理，导致消息丢失。

在创建 KafkaProducer 实例时，生产者应用会在后台创建并启动一个名为 Sender 的线程，该 Sender 线程开始运行时首先会创建与 Broker 的连接。Producer 启动时会发起与bootstrap.servers 参数设置的broker进行连接。

kafka的高性能IO主要表现在：

* 消息的批量处理
* 充分利用文件缓存（文件缓存的淘汰策略是LRU，kafka消息一般发送到broker就会立即被消费，文件胡缓存命中率较高）
* 磁盘顺序读写
* 零拷贝

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
* 默认分区策略：如果制指定了Key那么默认实现按消息键保序策略；如果没有指定则使用轮询

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

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1ggfaaxyq8rj30wu0e2dju.jpg" alt="截屏2020-07-04 下午9.57.36" style="zoom:50%;" />

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1ggfabmt532j30wa0ja0w7.jpg" alt="截屏2020-07-04 下午9.58.21" style="zoom:50%;" />

broker会在它所监听的每一个端口上运行一个Acceptor线程，这个线程会创建一个连接，并把它交给Processor线程去处理。Processor线程（也被叫作“网络线程”）的数量是可配置的。网络线程负责从客户端获取请求消息，把它们放进请求队列，然后从响应队列获取响应消息，把它们发送给客户端。请求队列是所有网络线程共享的，而响应队列则是每个网络线程专属的。Kafka 提供了 Broker 端参数 num.network.threads，用于调整该网络线程池的线程数，Broker 端参数 num.io.threads 控制了这个IO线程池中的线程数。Purgatory 组件是用来缓存延时请求（Delayed Request）的，所谓延时请求，就是那些一时未满足条件不能立刻处理的请求。

![截屏2020-01-04下午7.22.12](https://tva1.sinaimg.cn/large/006tNbRwgy1gakr0zwj1aj317q0u0why.jpg)

客户端发送请求前会发送元数据获取请求获得感兴趣主题所在的borker，客户端会缓存该元数据信息，每隔一段时间会发送一次元数据请求跟新客户端缓存，另外如果客户端收到非首领错误也会重新获取元数据。

在处理获取请求时Kafka使用<a href="#零复制">零复制</a>技术向客户端发送消息——也就是说，Kafka直接把消息从文件（或者更确切地说是Linux文件系统缓存）里发送到网络通道，而不需要经过任何中间缓冲区。

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

kafka中包含主副本和从副本，主副本对外提供服务，从副本不对外提供服务。

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

# 零复制

常规的文件读写流程如下：

![](https://tva1.sinaimg.cn/large/006tNbRwgy1gbjhx0kgkij30f409z0tq.jpg)

零复制的概念是避免将数据在内核空间和用户空间进行拷贝。

## mmap

应用程序调用`mmap()`，磁盘上的数据会通过`DMA`被拷贝的内核缓冲区，接着操作系统会把这段内核缓冲区与应用程序相关联，此时不进行任何数据拷贝。应用程序再调用`write()`时因为没有数据发生缺页中断，操作系统直接将内核缓冲区的内容拷贝到`socket`缓冲区中。这样就少了数据在内核空间和用户空间之间的数据拷贝。

# kafka数据压缩策略

kafka的消息层次分为两层：消息集合和消息

Producer 端压缩、Broker 端保持、Consumer 端解压缩。Kafka 会将启用了哪种压缩算法封装进消息集合中，这样当 Consumer 读取到消息集合时，它自然就知道了这些消息使用的是哪种压缩算法。

# 避免消息丢失的建议

* 不要使用 producer.send(msg)，而要使用 producer.send(msg, callback)。记住，一定要使用带有回调通知的 send 方法。
* 设置 acks = all。acks 是 Producer 的一个参数，代表了你对“已提交”消息的定义。如果设置成 all，则表明所有副本 Broker 都要接收到消息，该消息才算是“已提交”。这是最高等级的“已提交”定义。
* 设置 retries 为一个较大的值。这里的 retries 同样是 Producer 的参数，对应前面提到的 Producer 自动重试。当出现网络的瞬时抖动时，消息发送可能会失败，此时配置了 retries > 0 的 Producer 能够自动重试消息发送，避免消息丢失。
* 设置 unclean.leader.election.enable = false。这是 Broker 端的参数，它控制的是哪些 Broker 有资格竞选分区的 Leader。如果一个 Broker 落后原先的 Leader 太多，那么它一旦成为新的 Leader，必然会造成消息的丢失。故一般都要将该参数设置成 false，即不允许这种情况的发生。
* 设置 replication.factor >= 3。这也是 Broker 端的参数。其实这里想表述的是，最好将消息多保存几份，毕竟目前防止消息丢失的主要机制就是冗余。
* 设置 min.insync.replicas > 1。这依然是 Broker 端参数，控制的是消息至少要被写入到多少个副本才算是“已提交”。设置成大于 1 可以提升消息持久性。在实际环境中千万不要使用默认值 1。
* 确保 replication.factor > min.insync.replicas。如果两者相等，那么只要有一个副本挂机，整个分区就无法正常工作了。我们不仅要改善消息的持久性，防止数据丢失，还要在不降低可用性的基础上完成。推荐设置成 replication.factor = min.insync.replicas + 1。
* 确保消息消费完成再提交。Consumer 端有个参数 enable.auto.commit，最好把它设置成 false，并采用手动提交位移的方式。就像前面说的，这对于单 Consumer 多线程处理的场景而言是至关重要的。

# kafka拦截器

Kafka 拦截器分为生产者拦截器和消费者拦截器。生产者拦截器允许你在发送消息前以及消息提交成功后植入你的拦截器逻辑；而消费者拦截器支持在消费消息前以及提交位移后编写特定逻辑。这两种拦截器都支持链的方式，即你可以将一组拦截器串连成一个大的拦截器，Kafka 会按照添加顺序依次执行拦截器逻辑。

生产和消费端配置参数：interceptor.classes

实现接口：ProducerInterceptor和ConsumerInterceptor

# kafka幂等性

设置参数：enable.idempotence

它只能保证单分区上的幂等性，即一个幂等性 Producer 能够保证某个主题的一个分区上不出现重复消息，它无法实现多个分区的幂等性。其次，它只能实现单会话上的幂等性，不能实现跨会话的幂等性。这里的会话，你可以理解为 Producer 进程的一次运行。当你重启了 Producer 进程之后，这种幂等性保证就丧失了。

# kafka事务

目前主要是在 read committed 隔离级别上做事情。它能保证多条消息原子性地写入到目标分区，同时也能保证 Consumer 只能看到事务成功提交的消息。

设置事务型 Producer 的方法也很简单，满足两个要求即可：

* 和幂等性 Producer 一样，开启 enable.idempotence = true。
* 设置 Producer 端参数 transactional. id。最好为其设置一个有意义的名字。

因此在 Consumer 端，读取事务型 Producer 发送的消息也是需要一些变更的。修改起来也很简单，设置 isolation.level 参数的值即可。当前这个参数有两个取值：

* read_uncommitted
* read_committed

kafka生产者api提供了相应的事务操作函数。

# kafka控制器

Broker 在启动时，会尝试去 ZooKeeper 中创建 /controller 节点。Kafka 当前选举控制器的规则是：第一个成功创建 /controller 节点的 Broker 会被指定为控制器。

控制器指责：

* 主题管理
* 分区重分配
* Preferred领导者选举
* 集群成员管理
* 数据服务

在kafka集群运行的过程中只能有一台Broker充当控制器的角色，问了防止单点失效，当运行中的控制器突然宕机或意外终止时，Kafka 能够快速地感知到，并立即启用备用控制器来代替之前失败的控制器。控制器宕机时所有存活的Broker开始竞选新的控制器，竞选成功的会在zk上重建/controller节点，并从zk上读取集群元数据信息。

现在控制器采用单线程加事件队列的方案，引入了一个事件处理线程，统一处理各种控制器事件，然后控制器将原来执行的操作全部建模成一个个独立的事件，发送到专属的事件队列中，供此线程消费。

<img src="https://static001.geekbang.org/resource/image/b1/e5/b14c6f2d246cbf637f2fda5dae1688e5.png" alt="img" style="zoom:50%;" />

# kafka管理命令

```cmd
# 创建主题
bin/kafka-topics.sh --bootstrap-server broker_host:port --create --topic my_topic_name  --partitions 1 --replication-factor 1
# 删除主题
bin/kafka-topics.sh --bootstrap-server broker_host:port --delete  --topic <topic_name>
# 查询所有主题
bin/kafka-topics.sh --bootstrap-server broker_host:port --list
# 查询单个主题
bin/kafka-topics.sh --bootstrap-server broker_host:port --describe --topic <topic_name>
# 修改主题分区
bin/kafka-topics.sh --bootstrap-server broker_host:port --alter --topic <topic_name> --partitions <新分区数>
# 修改主题级别参数
bin/kafka-configs.sh --zookeeper zookeeper_host:port --entity-type topics --entity-name <topic_name> --alter --add-config max.message.bytes=10485760
# 生产消息
bin/kafka-console-producer.sh --broker-list kafka-host:port --topic test-topic --request-required-acks -1 --producer-property compression.type=lz4
# 消费消息
bin/kafka-console-consumer.sh --bootstrap-server kafka-host:port --topic test-topic --group test-group --from-beginning --consumer-property enable.auto.commit=false
# 测试生产者性能
bin/kafka-producer-perf-test.sh --topic test-topic --num-records 10000000 --throughput -1 --record-size 1024 --producer-props bootstrap.servers=kafka-host:port acks=-1 linger.ms=2000 compression.type=lz4
# 测试消费者性能
bin/kafka-consumer-perf-test.sh --broker-list kafka-host:port --messages 10000000 --topic test-topic
# 查看主题消息总数
bin/kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list kafka-host:port --time -2 --topic test-topic
# 查看消息文件数据
bin/kafka-dump-log.sh
# 查看消费组位移
bin/kafka-consumer-groups.sh
```

# 动态参数

Kafka对一些参数提供动态调整，将参数分为三类：

* read-only 只有重启broker修改才生效
* per-broker 动态参数，修改之后只会在对应的broker上生效
* cluster-wide 动态参数，修改之后对整个集群范围内有效

kafka将动态参数保存在zk的/config/brokers节点下，该 znode 下有两大类子节点。第一类子节点就只有一个，它有个固定的名字叫 < default >，保存的是前面说过的 cluster-wide 范围的动态参数；另一类则以 broker.id 为名，保存的是特定 Broker 的 per-broker 范围参数。

动态参数配置命令：

```cmd
# 集群范围内动态参数的配置
bin/kafka-configs.sh --bootstrap-server kafka-host:port --entity-type brokers --entity-default --alter --add-config unclean.leader.election.enable=true
# broker范围内动态参数的配置
bin/kafka-configs.sh --bootstrap-server kafka-host:port --entity-type brokers --entity-name 1 --alter --add-config unclean.leader.election.enable=false
# 删除集群范围内参数的配置
bin/kafka-configs.sh --bootstrap-server kafka-host:port --entity-type brokers --entity-default --alter --delete-config unclean.leader.election.enable
# 删除broker范围参数
bin/kafka-configs.sh --bootstrap-server kafka-host:port --entity-type brokers --entity-name 1 --alter --delete-config unclean.leader.election.enable
```

# kafka Stream

<img src="https://static001.geekbang.org/resource/image/2e/89/2ed121d8828da0dfa7cf1b5060f4d489.jpg" alt="img" style="zoom: 25%;" />

每个Kafka Streams实例都由一个消费者实例、特定的流处理逻辑，以及一个生产者实例组成，而这些实例中的消费者实例，共同构成了一个消费者组。

流在时间维度上聚合之后形成表，表在时间维度上不断更新形成流，这就是所谓的流表二元性。

# Kafka的zk数据存储



<img src="https://static001.geekbang.org/resource/image/80/b3/806ac0fc52ccbf50506e3b5d269b81b3.jpg" alt="img" style="zoom:50%;" />

圆角表示吧临时节点，左侧存储broker信息，右侧存储主题信息。每个分区节点下面是一个名为 state 的临时节点，节点中保存着分区当前的 leader 和所有的 ISR 的 BrokerID。这个 state 临时节点是由这个分区当前的 Leader Broker 创建的。如果这个分区的 Leader Broker 宕机了，对应的这个 state 临时节点也会消失，直到新的 Leader 被选举出来，再次创建 state 临时节点。