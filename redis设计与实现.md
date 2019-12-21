# 数据结构

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

压缩列表（ziplist）是列表和哈希的底层实现之一。当一个列表键只包含少量列表项，并且每个列表项要么就是小整数值，要么就是长度比较短的字符串，那么Redis就会使用压缩列表来做列表键的底层实现。

|  zlbytes   |         zltail         | zllen  | entry1 | entry2 | .... | entryN | zlend |
| :--------: | :--------------------: | :----: | :----: | :----: | :--: | :----: | :---: |
| 占用字节数 | 尾节点距起始节点字节数 | 节点数 |        |        |      |        | 0xFF  |

| previous_entry_length |    encoding    | content |
| :-------------------: | :------------: | :-----: |
|     上一节点长度      | 数据类型和长度 | 节点值  |

previous_entry_length属性的长度可以是1字节（小于254字节）或者5字节（大于等于254字节），所以可能存在连续跟新的问题，插入或者删除一个节点时可能导致previous_entry_length属性字段需要扩容，导致之后的所有节点都需要扩容。

## 对象

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

RDB持久化通过保存数据库中的键值对来记录数据库状态，AOF持久化通过保存REDIS服务器执行的写命令来记录数据库状态。

AOF实现包含以下三个步骤：

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

AOF文件的重写不是通过读取分析现有AOF文件来实现的，而是通过读取服务器数据库当前状态实现的。读取当前数据库现有数据然后用一条命令记录并存储。实现指令：BGREWRITEAOF。

通过创建子进程实现，在这期间服务器会创建一个重写缓冲区记录新到达的命令，子进程重写任务执行完后会向父进程发送信号，父进程会将重写缓冲区中的指令写入新的AOF文件。

# 事件

Redis服务器是一个事件驱动程序，服务器需要处理以下两类事件：

* 文件事件（网络请求）
* 时间事件（定时/周期性操作）

先执行文件事件，在执行时间事件。

## 文件事件

多个文件事件可能并发的出现，单I/O多路复用程序总会将所有产生的事件的套接字都放在一个队列中依次处理。I/O多路复用程序允许服务器同时监听套接字的AE_READABLE和AE_WRITABLE事件，如果一个套接字同时产生了这两种事件，那么会优先处理读事件。监听文件事件的最大阻塞时间是由到达时间最接近当前时间的时间事件决定的。

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

SLAVEOF或者设置slaveof选项

* 首次复制时从服务器向主服务器发送PSYNC命令
* 主服务器收到命令后执行BGSAVE生成RDB文件，并使用缓冲区记录从现在开始的所有写命令
* BGSAVE执行结束后，向从服务器发送RDB文件，从服务器载入
* 主服务器将缓冲区中的写命令发送给从服务器执行
* 往后主服务器每次执行写操作时会将该命令发送给从服务器

断线重连操作，主从双方会维护一个复制偏移量和服务器ID，主服务器会维护一个固定长度的先进先出队列（复制积压缓冲区），想从服务器发送写命令时会先将命令记录在该队列中。

* 从服务器重连时会发送之前保存的主服务器ID，如果该ID和当前连接的主服务器一样可以尝试进行部分重同步，否则执行完整重同步。
* 根据传给主服务器的偏移值进行判断是否执行部分重同步，如果偏移值之后（
  +1）的数据还在，则将缺少的命令传给从服务器，否则执行完整同步。

# 哨兵

启动方式：

```shell
redis-sentinel 配置文件路径
redis-server 配置文件路径 --sentinel
```

哨兵是通过连接主服务器获取相应从服务器的信息的，sentinel会将从服务器信息记录在主服务器实例结构的slaves字典里面，获取到从服务器信息后回创建到从服务器的连接，和主服务器一样创建命令连接和订阅连接（向_sentinel_:hello发送订阅信息）。默认会以十秒一次的频率向连接的服务器发送INFO命令获取服务器状态。每两秒一次通过命令连接向服务器的_sentinel_:hello频道发送信息。这也就是说，对于每个与Sentinel连接的服务器，Sentinel既通过命令连接向服务器的__sentinel__:hello频道发送信息，又通过订阅连接从服务器的__sentinel__:hello频道接收信息。

对于监视同一个服务器的多个Sentinel来说，一个Sentinel发送的信息会被其他Sentinel接收到，这些信息会被用于更新其他Sentinel对发送信息Sentinel的认知，也会被用于更新其他Sentinel对被监视服务器的认知。

每个sentinel服务器都会记录其他监视同一服务器的sentinel服务器的信息，双方会创建连接。