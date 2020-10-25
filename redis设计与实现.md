# 数据结构

redis支持的数据结构和底层实现：

 <img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1gj41kjne5ij30uk092q5f.jpg" alt="截屏2020-09-26 下午2.34.54" style="zoom:50%;" />

redis键值的组织结构：全局哈希表记录所有的键值对

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1gj41q5gdq8j30vc0fgwi8.jpg" alt="截屏2020-09-26 下午2.40.50" style="zoom:50%;" />

hash表操作会涉及到hash冲突和rehash带来的性能问题，redis采用渐进rehash。

##简单动态字符串

该结构是redis字符串的底层实现，redis并没有直接使用C字符串。SDS遵循C字符串以空字符结尾的惯例，保存空字符的1字节空间不计算在SDS的len属性里面，并且为空字符分配额外的1字节空间，以及添加空字符到字符串末尾等操作，都是由SDS函数自动完成的，所以这个空字符对于SDS的使用者来说是完全透明的。

```c
struct sdshdr{
  //记录buf数组中已使用字节的数量,等于SDS所保存字符串的长度
  //可以常数复杂度获取字符串的长度
  int len;
  //记录buf数组中未使用字节的数量
  int free;
  //字节数组，用于保存字符串
  char buf[];
};
```

SDS与C字符串的区别：

* 常数复杂度获取字符串长度
* 杜绝缓冲区溢出
* 减少字符串修改时内存重分配的次数（空间预分配，惰性空间释放）
* 忽略特殊字符的影响
* 兼容部分C字符串函数

String类型的底层存储结构：

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1gj9yzomt86j30vy0k879c.jpg" alt="截屏2020-10-01 下午5.39.35" style="zoom:50%;" />

redis会用一个redisObject对象存储数据类型，该对象包含元数据信息和实际的数据类型信息。根据存储类型大小的不同redis做了以上几种优化机制。除了以上的存储开销还有全局哈希表的存储开销，redis在分配内存时是按照2的次方来分配的，所以会存在占用的空间比实际存储的空间大。

<img src="/Users/chenjie/Library/Application Support/typora-user-images/截屏2020-10-01 下午5.43.00.png" alt="截屏2020-10-01 下午5.43.00" style="zoom:50%;" />

## 链表

Redis使用的C语言并没有内置链表实现，所以redis实现了自己的链表结构，是列表的底层实现之一，redis实现的列表是双端列表。

```c
typedef struct listNode{
  //前置节点
  struct listNode *prev;
  //后置节点
  struct listNode *next;
  //节点的值
  void *value;
}listNode;

typedef struct list{
  //表头节点
  listNode *head;
  //表尾节点
  listNode *tail;
  //链表所包含的节点数量
  unsigned long len;
  //节点值复制函数
  void *(*dup)(void *ptr);
  //节点值释放函数
  void (*free)(void *ptr);
  //节点值对比函数
  int (*match)(void *ptr,void *key);
}list;
```

## 字典

用于保存键值对的一种数据结构，字典中的每个键都是唯一的，redis本身就是一个键值对数据库，随意字典是redis数据库的底层实现，此外还是redis散列的底层实现之一，当一个哈希键包含的键值对比较多，又或者键值对中的元素都是比较长的字符串时，Redis就会使用字典作为哈希键的底层实现，redis的字典使用哈希表作为顶层实现。

```c
typedef struct dict{
  //用于操作键值对的函数
  dictType *type;
  //传输给上面函数的参数
  void *privdata;
  //哈希表
  //正常使用ht[0],当进行rehash时利用ht[1]进行扩容或收缩
  //rehash结束后会将ht[1]作为ht[0]
  dictht ht[2];
  //rehash索引
  //当rehash不在进行时，值为-1
  int rehashidx;
}dict;
```

Redis计算hash值和索引值的方法：

```c
hash=dict>type>hashFunction(key);
index=hash&dict>ht[x].sizemask;
```

当字典被用作数据库的底层实现，或者哈希键的底层实现时，Redis使用MurmurHash2算法来计算键的哈希值.

如果执行的是扩展操作，那么ht[1]的大小为第一个大于等于ht[0].used*2（2的n次方幂）

如果执行的是收缩操作，那么ht[1]的大小为第一个大于等于ht[0].used（2的n次方幂）

当以下条件中的任意一个被满足时，程序会自动开始对哈希表执行扩展操作：

* 服务器目前没有在执行BGSAVE命令或者BGREWRITEAOF命令，并且哈希表的负载因子大于等于1
* 服务器目前正在执行BGSAVE命令或者BGREWRITEAOF命令，并且哈希表的负载因子大于等于5。

负载因子计算公式：load_factor=ht[0].used/ht[0].size，负载因子小于0.1时会进行收缩。

redis的rehash操作是渐进的，不是一下子完成的，当进行rehash操作时，字典会同时持有ht[0]和ht[1]，在这期间除了添加操作都会在这两个哈希表上进行，rehashindx会记录rehash的进度。

## 跳跃表

Redis使用跳跃表作为有序集合的底层实现之一，如果一个有序集合包含的元素数量比较多，又或者有序集合中元素的成员（member）是比较长的字符串时，Redis就会使用跳跃表来作为有序集合键的底层实现。

```c
typedef struct zskiplistNode{
  //层
  struct zskiplistLevel{
    //前进指针
    struct zskiplistNode *forward;
    //跨度
    unsigned int span;
  }level[];
  //后退指针
  struct zskiplistNode *backward;
  //分值
  double score;
  //成员对象
  robj *obj;
}zskiplistNode;

typedef struct zskiplist{
  //表头节点和表尾节点
  struct zskiplistNode *header, *tail;
  //表中节点的数量
  unsigned long length;
  //表中层数最大的节点的层数
  int level;
}zskiplist;
  
```

## 整数集合

整数集合（intset）是集合的底层实现之一，当一个集合只包含整数值元素，并且这个集合的元素数量不多时，Redis就会使用整数集合作为集合键的底层实现。

```c
typedef struct intset{
  //编码方式
  //int16_t int32_t int64_t
  uint32_t encoding;
  //集合包含的元素数量
  uint32_t length;
  //保存元素的数组
  //数组中元素按从小到大的顺序排列
  //不包含重复元素
  //数组类型取决于编码方式的值
  int8_t contents[];
}intset;
```

### 升级

每当我们要将一个新元素添加到整数集合里面，并且新元素的类型比整数集合现有所有元素的类型都要长时，整数集合需要先进行升级（upgrade），然后才能将新元素添加到整数集合里面。

* 根据新元素的类型，扩展整数集合底层数组的空间大小，并为新元素分配空间。
* 将底层数组现有的所有元素都转换成与新元素相同的类型，并将类型转换后的元素放置到正确的位上，而且在放置元素的过程中，需要继续维持底层数组的有序性质不变。
* 将新元素添加到底层数组里面。

整数集合不支持降级操作，升级后编码就会保持升级后的状态。

## 压缩列表

压缩列表（ziplist）是列表，有序列表和哈希的底层实现之一。当一个列表键只包含少量列表项，并且每个列表项要么就是小整数值，要么就是长度比较短的字符串，那么Redis就会使用压缩列表来做列表键的底层实现。

|  zlbytes   |         zltail         | zllen  | entry1 | entry2 | .... | entryN | zlend |
| :--------: | :--------------------: | :----: | :----: | :----: | :--: | :----: | :---: |
| 占用字节数 | 尾节点距起始节点字节数 | 节点数 |        |        |      |        | 0xFF  |

| previous_entry_length |    encoding    | content |
| :-------------------: | :------------: | :-----: |
|     上一节点长度      | 数据类型和长度 | 节点值  |

previous_entry_length属性的长度可以是1字节（小于254字节）或者5字节（大于等于254字节），所以可能存在连续跟新的问题，插入或者删除一个节点时可能导致previous_entry_length属性字段需要扩容，导致之后的所有节点都需要扩容。

## 对象（RedisObject）

```c
typedef struct redisObject{
  //类型
  unsigned type;
  //编码
  //具体的底层实现数据结构
  unsigned encoding;
  //指向底层实现数据结构的指针
  void *ptr;
  //引用计数
  //用于内存回收，内存共享
  //redis在初始化时会创建10000个字符串对象，包含从0到9999的所有整数用于共享
  //OBJECT REFCOUNT可查看
  int refcount;
  //最后一次被访问的时间
  //OBJECT IDLETIME查看空转时长=当前时间-lru
  //如果服务器打开了maxmemory选项，并且回收内存的算法为volatile-lru或者allkeys-lru，当服务器占用的内存超过了maxmemory选项所设置的上限值时，空转时长较高的键会优先被释放
  unsigned lru;
}robj;
```

### 对象类型

使用type命令可以查看对象类型

* REDIS_STRING（字符串对象）
* REDIS_LIST（列表对象）
* REDIS_HASH（哈希对象）
* REDIS_SET（集合对象）
* REDIS_ZSET（有序集合对象）

| 类型         | 编码                      |
| :----------- | :------------------------ |
| REDIS_STRING | REDIS_ENCODING_INT        |
| REDIS_STRING | REDIS_ENCODING_EMBSTR     |
| REDIS_STRING | REDIS_ENCODING_RAW        |
| REDIS_LIST   | REDIS_ENCODING_ZIPLIST    |
| REDIS_LIST   | REDIS_ENCODING_LINKEDLIST |
| REDIS_HASH   | REDIS_ENCODING_LINKEDLIST |
| REDIS_HASH   | REDIS_ENCODING_HT         |
| REDIS_SET    | REDIS_ENCODING_INTSET     |
| REDIS_SET    | REDIS_ENCODING_HT         |
| REDIS_ZSET   | REDIS_ENCODING_ZIPLIST    |
| REDIS_ZSET   | REDIS_ENCODING_SKIPLIST   |

## GEO

GEO可以用来处理位置信息，底层采用有序列表进行存储，通过经纬度进行排序。但是一般经纬度是一组数据不能进行有效的排序，所以GEO采用GeoHash对经纬度进行编码（通过类似哈夫曼编码的方式分别对经纬度进行编码，然后合并）。常用的操作指令如下：

```cmd
// 添加经纬度元素信息
GEOADD cars:locations 116.034579 39.030452 33
// 查询给定经纬度一定范围内的元素
GEORADIUS cars:locations 116.054579 39.030452 5 km ASC COUNT 10
```

## 自定义数据类型

需要的时候自己看

## RedisTimeSeries

需要的时候自己看（存储时间序列）

# 数据库

```c
struct redisServer{
  //...
  //一个数组，保存着服务器中的所有数据库
  //默认初始化时创建16个数据库
  //可以通过服务器配置的database选项进行配置
  //selct可以切换数据库
  redisDb *db;
  //服务器的数据库数量
  int dbnum;
};

typedef struct redisClient{
  //...
  //记录客户端当前正在使用的数据库
  redisDb *db;
  //...
}redisClient;

typedef struct redisDb{
  //...
  //数据库键空间，保存着数据库中的所有键值对
  dict *dict;
  //过期字典
  dict *expires;
}redisDb;
```

在读取一个键之后（读操作和写操作都要对键进行读取），服务器会根据键是否存在来更新服务器的键空间命中（hit）次数或键空间不命中（miss）次数，这两个值可以在INFOstats命令的keyspace_hits属性和keyspace_misses属性中查看。

设置键的过期时间的方法：

* EXPIRE<key><ttl>命令用于将键key的生存时间设置为ttl秒。

* PEXPIRE<key><ttl>命令用于将键key的生存时间设置为ttl毫秒。

* EXPIREAT<key><timestamp>命令用于将键key的过期时间设置为timestamp所指定的秒数时间戳。

* PEXPIREAT<key><timestamp>命令用于将键key的过期时间设置为timestamp所指定的毫秒数时间戳。

  上面的三种命令最后都会转换为该命令进行实现

* PERSIST命令可以移除过期时间

redis的过期删除策略：惰性删除和定期删除。

在执行SAVE命令或者BGSAVE命令创建一个新的RDB文件时，程序会对数据库中的键进行检查，已过期的键不会被保存到新创建的RDB文件中。

在启动Redis服务器时，如果服务器开启了RDB功能，那么服务器将对RDB文件进行载入。如果以主服务器模式运行那么会忽略过期键，以从服务器模式运行不论过期与不过期。

当服务器以AOF持久化模式运行时，如果数据库中的某个键已经过期，但它还没有被惰性删除或者定期删除，那么AOF文件不会因为这个过期键而产生任何影响。当过期键被惰性删除或者定期删除之后，程序会向AOF文件追加一条DEL命令，来显式地记录该键已被删除。

数据库通知是Redis2.8版本新增加的功能，这个功能可以让客户端通过订阅给定的频道或者模式，来获知数据库中键的变化，以及数据库中命令的执行情况。服务器配置的notify-keyspace-events选项决定了服务器所发送通知的类型：

* 想让服务器发送所有类型的键空间通知和键事件通知，可以将选项的值设置为AKE。
* 想让服务器发送所有类型的键空间通知，可以将选项的值设置为AK。
* 想让服务器发送所有类型的键事件通知，可以将选项的值设置为AE。
* 想让服务器只发送和字符串键有关的键空间通知，可以将选项的值设置为K$。
* 想让服务器只发送和列表键有关的键事件通知，可以将选项的值设置为El。

# RDB持久化

## RDB的创建与载入

RDB记录的是某一时刻的数据，所以**具备快速恢复的优势**。

RDB持久化既可以手动执行，也可以根据服务器配置选项定期执行，该功能可以将某个时间点上的数据库状态保存到一个RDB文件中，有两个Redis命令可以用于生成RDB文件，一个是SAVE，另一个是BGSAVE。

SAVE命令会阻塞Redis服务器进程，直到RDB文件创建完毕为止，在服务器进程阻塞期间，服务器不能处理任何命令请求；BGSAVE命令会派生出一个子进程，然后由子进程负责创建RDB文件，服务器进程（父进程）继续处理命令请求。RDB文件的载入工作是在服务器启动时自动执行的，如果服务器开启了AOF持久化功能，那么服务器会优先使用AOF文件来还原数据库状态，因为一般AOF文件的更新频率比RDB文件高。

在BGSAVE命令执行期间，服务器会拒绝客户端传过来的SAVE和BGSAVE命令。如果BGSAVE命令正在执行，那么客户端发送的BGREWRITEAOF命令会被延迟到BGSAVE命令执行完毕之后执行。如果BGREWRITEAOF命令正在执行，那么客户端发送的BGSAVE命令会被服务器拒绝。

服务器在载入RDB文件时会一直处于阻塞状态。

通过配置save选项可以使服务器定期执行BGSAVE命令，例如：save 900 1表示服务器在900秒内对数据库进行了至少一次修改。可以配置多个save选项，可要其中一个被满足就会执行。如果没有主动设置会有默认值。

```c
struct redisServer{
  //...
  //记录了保存条件的数组
  struct saveparam *saveparams;
  //上次保存后修改次数
  long long dirty;
  //上一次执行保存的时间
  time_t lastsave;
  //...
};

struct saveparam{
  //秒数
  time_t seconds;
  //修改数
  int changes;
};
```

Redis的服务器周期性操作函数serverCron默认每隔100毫秒就会执行一次，该函数用于对正在运行的服务器进行维护，它的其中一项工作就是检查save选项所设置的保存条件是否已经满足，如果满足的话，就执行BGSAVE命令。

RDB的COW机制：在实际执行过程中，是子进程复制了主线程的页表，所以通过页表映射，能读到主线程的原始数据，而当有新数据写入或数据修改时，主线程会把新数据或修改后的数据写到一个新的物理内存地址上，并修改主线程自己的页表映射。所以，子进程读到的类似于原始数据的一个副本，而主线程也可以正常进行修改。

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1gj8lv3sq76j30vy0i2dj7.jpg" alt="截屏2020-09-30 下午1.19.51" style="zoom:50%;" />

## RDB文件结构

文件结构：（可以通过od命令读取RDB文件）

* REDIS 

  常量用于标识RDB文件

* db_version

  4字节，它的值是一个字符串表示的整数

* databases

  包含零个或任意多个数据库，以及各个数据库中的键值对数据

* EOF

* check_sum

  8字节的无符号整数，通过前面四个部分计算出

databases部分：

* SELECTDB

* db_number

  当载入程序读入该字段时服务器会调用select命令进行数据库的切换

* key_value_pairs

key_value_pairs:

* EXPIRETIME_MS

  具有过期时间的键值对才有，1字节，告知读入程序接下来读入的将是一个以毫秒为单位的过期时间。

* ms

  同上，过期时间

* TYPE

  REDIS_RDB_TYPE_STRING

  REDIS_RDB_TYPE_LIST

  REDIS_RDB_TYPE_SET

  REDIS_RDB_TYPE_ZSET

  REDIS_RDB_TYPE_HASH

  REDIS_RDB_TYPE_LIST_ZIPLIST

  REDIS_RDB_TYPE_SET_INTSET

  REDIS_RDB_TYPE_ZSET_ZIPLIST

  REDIS_RDB_TYPE_HASH_ZIPLIST

* key

* value

# AOF持久化

## AOF文件创建

写AOF文件是主线程完成的，虽然由于是在命令执行后写文件不会阻塞当前命令，但因为主线程操作会影响后续的命令执行。

RDB持久化通过保存数据库中的键值对来记录数据库状态，AOF持久化通过保存REDIS服务器执行的写命令来记录数据库状态。

AOF实现包含以下三个步骤（主线程执行）：

（先执行命令，把数据写入内存，然后才记录日志。redis在写日志的时候并不会对写入的命令做语法检查，所以如果在执行命令前写日志可能会导致写入错误的命令）

* 命令追加

  当服务器执行完一个写命令之后会向aof_buf缓冲区追加该命令

* 文件写入

  Redis服务器就是一个事件循环，每次结束一个事件循环之前都会考虑是否需要将aof_buf中的内容写入和同步到AOF文件中。是否进行同步操作取决于服务器配置appendfsync：

  * always
  * everysec 如果距离上次同步超过1秒则执行同步操作
  * no

  写入是指下入文件缓存，同步是指同步到磁盘。

* 文件同步

## AOF文件重写

AOF重写是通过fork子进程执行的，这样是为了防止阻塞主线程，但是fork操作本身是一次系统调用会阻塞当前线程，fork子进程需要拷贝必要的数据结构，这个拷贝时长取决于实例的内存大小等因素。

AOF文件的重写不是通过读取分析现有AOF文件来实现的，而是通过读取服务器数据库当前状态实现的。读取当前数据库现有数据然后用一条命令记录并存储。实现指令：BGREWRITEAOF。重写机制具有多变一的功能，可以使日志文件变小

通过创建子进程实现，在这期间服务器会创建一个重写缓冲区记录新到达的命令，子进程重写任务执行完后会向父进程发送信号，父进程会将重写缓冲区中的指令写入新的AOF文件。

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1gj45wngw7uj30vc0ektc2.jpg" alt="截屏2020-09-26 下午5.05.22" style="zoom:50%;" />

## 混合持久化机制

Redis 4.0 中提出了一个混合使用 AOF 日志和内存快照的方法。简单来说，内存快照以一定的频率执行，在两次快照之间，使用 AOF 日志记录这期间的所有命令操作。

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1gj8m0hr23tj30vm0jq434.jpg" alt="截屏2020-09-30 下午1.25.00" style="zoom:50%;" />

# 事件

Redis服务器是一个事件驱动程序，服务器需要处理以下两类事件：

* 文件事件（网络请求）
* 时间事件（定时/周期性操作）

先执行文件事件，在执行时间事件。

## 文件事件

多个文件事件可能并发的出现，单I/O多路复用程序总会将所有产生的事件的套接字都放在一个队列中依次处理。I/O多路复用程序允许服务器同时监听套接字的AE_READABLE和AE_WRITABLE事件，如果一个套接字同时产生了这两种事件，那么会优先处理读事件。监听文件事件的最大阻塞时间是由到达时间最接近当前时间的时间事件决定的。

![截屏2020-01-04下午7.09.10](https://tva1.sinaimg.cn/large/006tNbRwgy1gakqowz0ydj31240qowhi.jpg)

## 时间事件

服务器将所有时间事件都放在一个无序链表中（不按时间排序），每当时间事件执行器运行时，遍历整个链表，查找所有已到达的时间事件并调用相应的事件处理器。

## 客户端关闭

这里的客户端信息指的都是服务器端保存的，和具体的客户端没什么联系，下面列举的客户端属性也都是指的服务器端的。

* 客户端进程被杀死
* 客户端向服务器发送了不符合协议格式的命令
* client kill
* 服务器设置了timeout，主从服务器另说
* 客户端发送请求大小超过输入缓冲区（这里的缓冲区指的是服务器端内存）
* 发送给客户端的命令回复大小超过了输出缓冲区的大小（这里的缓冲区同上）

客户端分为伪客户端和普通客户端，伪客户端命令来源于本地不是网络，例如lua脚本获取AOF文件（载入AOF文件时会创建用于执行AOF文件中命令的伪客户端），伪客户端的套接字描述符为-1。

# 主从复制

SLAVEOF或者设置slaveof选项（redis5.0之后使用replicaof）

* 首次复制时从服务器向主服务器发送PSYNC命令
* 主服务器收到命令后执行BGSAVE生成RDB文件，并使用缓冲区记录从现在开始的所有写命令
* BGSAVE执行结束后，向从服务器发送RDB文件，从服务器载入
* 主服务器将缓冲区中的写命令发送给从服务器执行
* 往后主服务器每次执行写操作时会将该命令发送给从服务器

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1gj8mw2k32gj30v60e0jvi.jpg" alt="截屏2020-09-30 下午1.55.21" style="zoom:50%;" />

断线重连操作，主从双方会维护一个复制偏移量和服务器ID，主服务器会维护一个固定长度的先进先出队列（复制积压缓冲区），想从服务器发送写命令时会先将命令记录在该队列中。

* 从服务器重连时会发送之前保存的主服务器ID，如果该ID和当前连接的主服务器一样可以尝试进行部分重同步，否则执行完整重同步。
* 根据传给主服务器的偏移值进行判断是否执行部分重同步，如果偏移值之后（
  +1）的数据还在，则将缺少的命令传给从服务器，否则执行完整同步。

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1gj8mwnh750j30v20fydjs.jpg" alt="截屏2020-09-30 下午1.55.56" style="zoom:50%;" />

repl_backlog_buffer：主库的所有写命令都会记录在该内存区域，用来同步主从之间的差异。

replication buffer：网络缓冲区。

# 哨兵

启动方式：

```shell
redis-sentinel 配置文件路径
redis-server 配置文件路径 --sentinel
```

## 哨兵初始化

哨兵是通过连接主服务器获取相应从服务器的信息的，sentinel会将从服务器信息记录在主服务器实例结构的slaves字典里面，获取到从服务器信息后回创建到从服务器的连接，和主服务器一样创建命令连接和订阅连接（向_sentinel_:hello发送订阅信息）。默认会以十秒一次的频率向连接的服务器发送INFO命令获取服务器状态。每两秒一次通过命令连接向服务器的_sentinel_:hello频道发送信息。这也就是说，对于每个与Sentinel连接的服务器，Sentinel既通过命令连接向服务器的__sentinel__:hello频道发送信息，又通过订阅连接从服务器的__sentinel__:hello频道接收信息。

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1gj8zcufxhrj30v00hmn0m.jpg" alt="截屏2020-09-30 下午9.06.44" style="zoom:50%;" />

对于监视同一个服务器的多个Sentinel来说，一个Sentinel发送的频道信息会被其他Sentinel接收到，这些信息会被用于更新其他Sentinel对发送信息Sentinel的认知，也会被用于更新其他Sentinel对被监视服务器的认知。

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1gj8zasaa7sj30vc0h8wic.jpg" alt="截屏2020-09-30 下午9.04.41" style="zoom:50%;" />

每个sentinel服务器都会记录其他监视同一服务器的sentinel服务器的信息，双方会创建命令连接。

通过订阅哨兵的指定频道我们可以获取哨兵的运行状态：

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1gj8zg3xvhqj30uw0j0dmn.jpg" alt="截屏2020-09-30 下午9.09.51" style="zoom:50%;" />

初始化哨兵时不会向初始化一般redis服务器一样载入RDB或者AOF文件，载入的命令表也不一样。

```c
struct sentinelState{
  //当前纪元，用于实现故障转移
  uint64_t current_epoch;
  //保存了所有被这个sentinel监视的主服务器
  dict *masters;
  //是否进入了TILT模式？
  inttilt;
  //目前正在执行的脚本的数量
  int running_scripts;
  //进入TILT模式的时间
  mstime_t tilt_start_time;
  //最后一次执行时间处理器的时间
  mstime_t previous_time;
  //一个FIFO队列，包含了所有需要执行的用户脚本
  list *scripts_queue;
}sentinel;
```

## 检测主观下线

默认情况下，Sentinel会以每秒一次的频率向所有与它创建了命令连接的实例发送ping命令来判断实例是否在线。Sentinel配置文件中的down-after-milliseconds选项指定了Sentinel判断实例进入主观下线所需的时间长度：如果一个实例在down-after-milliseconds毫秒内，连续向Sentinel返回无效回复，那么Sentinel会修改这个实例所对应的实例结构，在结构的flags属性中打开SRI_S_DOWN标识，以此来表示这个实例已经进入主观下线状态。

用户设置的down-after-milliseconds选项的值，不仅会被Sentinel用来判断主服务器的主观下线状态，还会被用于判断主服务器属下的所有从服务器，以及所有同样监视这个主服务器的其他Sentinel的主观下线状态。

当前Sentinel判断一服务器为主观下线后，会询问同样监视这一服务器的其他Sentinel，如果足够数量的Sentinel认为该服务器下线了，那么当前Sentinel会判断该服务器客观下线，并对该服务器进行故障转移。

## 选举领头Sentinel

当一个主服务器被判断为客观下线时，监视这个下线主服务器的各个Sentinel会进行协商，选举出一个领头Sentinel，并由领头Sentinel对下线主服务器执行故障转移操作。

```shell
SENTINEL is-master-down-by-addr <ip> <port> <current_epoch> <runid>
```

current_epoch:计数器，每进行一个选举就+1

runid:运行id

回复信息：

1)<down_state>

2)<leader_runid>

3)<leader_epoch>

选举流程：

* 服务器被判定客观下线，向其他Sentinel服务器发送SENTINEL is-master-down-by-addr命令
* 收到命令的服务器如果没有设置局部领头Sentinel，会将命令中的runid设置为当前局部领头并回复
* 收到回复后进行统计，获得超过半数的成为领头

## 故障转移

* 从下线的主服务器的从服务器中选出新的主服务器
* 其他从服务器复制新的主服务器
* 下线主服务器设置为新主服务器的从服务器

# 集群

## 集群节点

向一个节点node发送CLUSTER MEET命令，可以让node节点与ip和port所指定的节点进行握手（handshake），当握手成功时，node节点就会将ip和port所指定的节点添加到node节点当前所在的集群中。

```shell
CLUSTER MEET <ip> <port>
CLUSTER NODES //查看当前集群包含的节点
CLUSTER INFO  //查看集群相关信息
```

一个节点就是一个运行在集群模式下的Redis服务器，Redis服务器在启动时会根据cluster-enabled配置选项是否为yes来决定是否开启服务器的集群模式。![截屏2019-12-24下午9.42.29](https://tva1.sinaimg.cn/large/006tNbRwgy1gaagrhltt3j310s0a0myr.jpg)

之后，节点A会将节点B的信息通过Gossip协议传播给集群中的其他节点，让其他节点也与节点B进行握手，最终，经过一段时间之后，节点B会被集群中的所有节点认识。

## 槽指派

Redis集群通过分片的方式来保存数据库中的键值对：集群的整个数据库被分为16384个槽（slot），数据库中的每个键都属于这16384个槽的其中一个，集群中的每个节点可以处理0个或最多16384个槽。当数据库中的16384个槽都有节点在处理时，集群处于上线状态（ok）；相反地，如果数据库中有任何一个槽没有得到处理，那么集群处于下线状态（fail）。

通过向节点发送CLUSTERADDSLOTS命令，我们可以将一个或多个槽指派（assign）给节点负责：

```shell
CLUSTER ADDSLOTS <slot> [slot...]
CLUSTER KEYSLOT <key> //查看给定键属于哪个槽
```

```c
struct clusterNode{
  //...
  unsigned char slots[16384/8];
  int numslots;
  //...
};
```

slots属性是一个二进制位数组（bitarray），这个数组的长度为16384/8=2048个字节，共包含16384个二进制位。Redis以0为起始索引，16383为终止索引，对slots数组中的16384个二进制位进行编号，并根据索引i上的二进制位的值来判断节点是否负责处理槽i：

* 如果slots数组在索引i上的二进制位的值为1，那么表示节点负责处理槽i。
* 如果slots数组在索引i上的二进制位的值为0，那么表示节点不负责处理槽i。

节点会将自己负责的slot信息发送给其他节点，所以集群中的每个节点都会知道数据库中的16384个槽分别被指派给了集群中的哪些节点。

此外节点的clusterState结构中的slots数组记录了集群中所有16384个槽的指派信息：

* 如果slots[i]指针指向NULL，那么表示槽i尚未指派给任何节点。
* 如果slots[i]指针指向一个clusterNode结构，那么表示槽i已经指派给了clusterNode结构所代表的节点。

计算键属于哪个槽：

```c
CRC16(key) & 16383
```

一个集群客户端通常会与集群中的多个节点创建套接字连接，而所谓的节点转向实际上就是换一个套接字来发送命令。如果客户端尚未与想要转向的节点创建套接字连接，那么客户端会先根据MOVED错误提供的IP地址和端口号来连接节点，然后再进行转向。

节点只使用0号数据库，此外除了将键值对保存在数据库里面之外，节点还会用clusterState结构中的slots_to_keys跳跃表来保存槽和键之间的关系，方便对于属于某个或者某些槽的所有键进行批量操作，命令CLUSTER GETKEYSINSLOT <slot> <count>命令可以返回最多count个属于槽slot的数据库键，而这个命令就是通过遍历slots_to_keys跳跃表来实现的。

## 重新分片

redis-trib对集群的单个槽slot进行重新分片的步骤如下：

* redis-trib对目标节点发送CLUSTER SETSLOT<slot>IMPORTING<source_id>命令，让目标节点准备好从源节点导入（import）属于槽slot的键值对。
* redis-trib对源节点发送CLUSTER SETSLOT<slot>MIGRATING<target_id>命令，让源节点准备好将属于槽slot的键值对迁移（migrate）至目标节点。
* redistrib向源节点发送CLUSTER GETKEYSINSLOT<slot><count>命令，获得最多count个属于槽slot的键值对的键名（keyname）。
* 对于步骤3获得的每个键名，redis-trib都向源节点发送一个MIGRATE<target_ip><target_port><key_name>0<timeout>命令，将被选中的键原子地从源节点迁移至目标节点。
* 重复执行步骤3和步骤4，直到源节点保存的所有属于槽slot的键值对都被迁移至目标节点为止。
* redistrib向集群中的任意一个节点发送CLUSTER SETSLOT<slot>NODE<target_id>命令，将槽slot指派给目标节点，这一指派信息会通过消息发送至整个集群，最终集群中的所有节点都会知道槽slot已经指派给了目标节点。

如果分片设计多个槽，重复执行以上操作。

当客户端向源节点发送一个与数据库键有关的命令，并且命令要处理的数据库键恰好就属于正在被迁移的槽时：

* 源节点会先在自己的数据库里面查找指定的键，如果找到的话，就直接执行客户端发送的命令。
* 如果源节点没能在自己的数据库里面找到指定的键，那么这个键有可能已经被迁移到了目标节点，源节点将向客户端返回一个ASK错误，指引客户端转向正在导入槽的目标节点，并再次发送之前想要执行的命令。

```c
typedef struct clusterState{
  //...
  //当前节点正在从别的节点导入的槽
  //importing_slots_from[i]指向clusterNode结构，那么表示当前节点正在从clusterNode所代表的节点导入槽i
  //CLUSTER SETSLOT<slot>IMPORTING<source_id>实现依据
  clusterNode *importing_slots_from[16384];
  //当前节点正在迁移至其他节点的槽
  //CLUSTER SETSLOT<slot>MIGRATING<target_id>实现依据
  clusterNode *migrating_slots_to[16384];
  //...
}clusterState;
```

当客户端接收到ASK错误并转向至正在导入槽的节点时，客户端会先向节点发送一个ASKING命令，然后才重新发送想要执行的命令，这是因为如果客户端不发送ASKING命令，而直接发送想要执行的命令的话，那么客户端发送的命令将被节点拒绝执行，并返回MOVED错误。ASKING命令唯一要做的就是打开发送该命令的客户端的REDIS_ASKING标识，在一般情况下，如果客户端向节点发送一个关于槽i的命令，而槽i又没有指派给这个节点的话，那么节点将向客户端返回一个MOVED错误；但是，如果节点的clusterState.importing_slots_from[i]显示节点正在导入槽i，并且发送命令的客户端带有REDIS_ASKING标识，那么节点将破例执行这个关于槽i的命令一次。客户端的REDIS_ASKING标识是一个一次性标识，当节点执行了一个带有REDIS_ASKING标识的客户端发送的命令之后，客户端的REDIS_ASKING标识就会被移除。

MOVED命令和ASK命令的区别在于ASK命令并不会更新客户端缓存的哈希槽的分配信息。

## 复制与故障转移

```shell
CLUSTER REPLICATE <node_id> //设置从节点命令,发送完开始进行复制（同之前）
```

```c
struct clusterNode{
  //...
  //如果这是一个从节点，那么指向主节点
  struct clusterNode *slaveof;
  //...
};
```

一个节点成为从节点，并开始复制某个主节点这一信息会通过消息发送给集群中的其他节点，最终集群中的所有节点都会知道某个从节点正在复制某个主节点。集群中的所有节点都会在代表主节点的clusterNode结构的slaves属性和numslaves属性中记录正在复制这个主节点的从节点名单。

集群中的每个节点都会定期地向集群中的其他节点发送PING消息，以此来检测对方是否在线，如果接收PING消息的节点没有在规定的时间内，向发送PING消息的节点返回PONG消息，那么发送PING消息的节点就会将接收PING消息的节点标记为疑似下线。当一个主节点A通过消息得知主节点B认为主节点C进入了疑似下线状态时，主节点A会在自己的clusterState.nodes字典中找到主节点C所对应的clusterNode结构，并将主节点B的下线报告（failure report）添加到clusterNode结构的fail_reports链表里面。如果在一个集群里面，半数以上负责处理槽的主节点都将某个主节点x报告为疑似下线，那么这个主节点x将被标记为已下线（FAIL），将主节点x标记为已下线的节点会向集群广播一条关于主节点x的FAIL消息，所有收到这条FAIL消息的节点都会立即将主节点x标记为已下线。

故障转移和主节点的选举和哨兵类似。

# 发布订阅

```shell
publish
subscribe
//如果有一个或多个模式pattern与频道channel相匹配，那么将消息message发送给pattern模式的订阅者
psubscribe //订阅多个主题，利用模式（类似正则表达式）
pubsub channels [pattern]//返回当前被订阅频道
PUBSUB NUMSUB [channel-1 channel-2...channel-n] //返回给定频道的订阅者数量
PUBSUB NUMPAT //返回服务器当前被订阅模式的数量
```

```c
struct redisServer{
  //...
  //保存所有频道的订阅关系
  //键为订阅频道，值为订阅客户端链表
  dict *pubsub_channels;
  //保存所有模式订阅关系
  list *pubsub_patterns;
  //...
};

typedef struct pubsubPattern{
  //订阅模式的客户端
  redisClient *client;
  //被订阅的模式
  robj *pattern;
}pubsubPattern;
```

# 事务

```shell
MULTI
EXEC
WATCH

```

所有对数据库进行修改的命令在执行前都会查看是否有客户端正在监视刚刚被命令修改的键，如果有那么将监视的客户端标识REDIS_DIRTY_CAS，服务器在接收到EXEC命令时会进行下面的判断。

![截屏2019-12-29下午8.44.34](https://tva1.sinaimg.cn/large/006tNbRwgy1gadvow3vl1j313y0imjte.jpg)

在Redis中，事务总是具有原子性（Atomicity）、一致性（Consistency）和隔离性（Isolation），并且当Redis运行在某种特定的持久化模式下时，事务也具有耐久性（Durability）。

Redis不支持事务回滚机制（rollback），即使事务队列中的某个命令在执行期间出现了错误，整个事务也会继续执行下去，直到将事务队列中的所有命令都执行完毕为止。

# 排序

```shell
SORT <key> //对包含数字值的键进行排序，默认升序，可用ASC（升）/DESC（降）指定
SORT <key> ALPHA //对包含字符串值的键进行排序
MSET apple-price 8 banana-price 5.5 cherry-price 7 //设置权重键
SORT fruits BY *-price //按给定权值排序
SORT fruits BY 【】 ALPHA //设置的键的权重值为字符串
LIMIT <offset> <count> //加在排序命令后面
GET *-name（例子） //返回排序结果对应的其他值，后面的参数用于指定返回的值
STORE <key> //默认不存储排序结果，该命令加在后面表示将排序结果存在给定的键中
```

排序选项执行顺序：

* 排序
* 限制结果集长度
* 获取外部键
* 保存排序结果集
* 返回排序结果

# 二进制数组

```shell
SETBIT
GETBIT
BITCOUNT //统计1的个数
BITOP //多个二进制数组进行运算
```

# 慢查询日志

* slowlog-log-slower-than //指定执行时间超过多少微秒的命令会被记录
* slowlog-max-len //指定服务器最多保存多少条慢查询日志

可通过CONFIG SET命令对上面的参数进行设置，SLOWLOG GET命令获取记录的慢查询日志

# 监视器

通过执行MONITOR命令，客户端可以将自己变为一个监视器，实时地接收并打印出服务器当前处理的命令请求的相关信息。

# redis网络模型

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1gj4463simnj30vu0ko0xm.jpg" alt="截屏2020-09-26 下午4.05.23" style="zoom:50%;" />

# redis实践

## 集合统计模式

* 聚合统计 //多个集合数据之间的统计比较
* 排序统计
* 二值状态统计 //存储的值不就是1就是0，可以用Bitmap
* 基数统计

## Redis消息队列方案

消息队列需求满足的条件：

* 消息保序
* 重复消息处理
* 消息可靠性保证

### 基于List的消息队列

* List本身就是按照插入顺序进行存储
* 生产者提供唯一性标识
* BRPOPLPUSH 命令从一个 List 中读取消息，同时Redis 会把这个消息插入到另一个 List 留存

### 基于Stream的消息队列解决方案

Streams 是 Redis 专门为消息队列设计的数据类型，它提供了丰富的消息队列操作命令。

* XADD：插入消息，保证有序，可以自动生成全局唯一 ID；
* XREAD：用于读取消息，可以按 ID 读取数据；
* XREADGROUP：按消费组形式读取消息，队列中的消息被消费组中的一个消费者读取后就不会被组内的其他消费者读取
* XPENDING 和 XACK：XPENDING 命令可以用来查询每个消费组内所有消费者已读取但尚未确认的消息，而 XACK 命令用于向消息队列确认消息处理已完成。

为了保证消费者在发生故障或宕机再次重启后，仍然可以读取未处理完的消息，Streams 会自动使用内部队列（也称为 PENDING List）留存消费组里每个消费者读取的消息，直到消费者使用 XACK 命令通知 Streams“消息已经处理完成”。如果消费者没有成功处理消息，它就不会给 Streams 发送 XACK 命令，消息仍然会留存。此时，消费者可以在重启后，用 XPENDING 命令查看已读取、但尚未确认处理完成的消息。

## Redis性能影响

### Redis交互操作

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1gjazan78ubj30v20pqdlf.jpg" alt="截屏2020-10-02 下午2.35.45" style="zoom:50%;" />

### 主流CPU架构（NUMA）

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1gjazzvhcusj30v20gsq70.jpg" alt="截屏2020-10-02 下午2.59.57" style="zoom:50%;" />

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1gjb009p8x7j30us0d0wgb.jpg" alt="截屏2020-10-02 下午3.00.22" style="zoom:50%;" />

如果应用程序先在一个 Socket 上运行，并且把数据保存到了内存，然后被调度到另一个 Socket 上运行，此时，应用程序再进行内存访问时，就需要访问之前 Socket 上连接的内存，这种访问属于远端内存访问。和访问 Socket 直接连接的内存相比，远端内存访问会增加应用程序的延迟。这种情况我们可以通过taskset命令将redis实例绑定在一个CPU核上运行。

通过以下命令可以检测当前redis的基础性能：./redis-cli --intrinsic-latency 120，将这个基础性能作为参照来判断redis是否响应慢了。

