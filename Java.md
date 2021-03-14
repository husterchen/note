# Java 8 主要改变

* Stream API
* 方法引用（行为参数化）
* 无状态任务的并行处理
* Lambda表达式
* 谓词参数
* 更好的利用多核并行操作
* 接口默认方法
* Optional

# 行为参数化

行为参数化的好处在于你可以把迭代要筛选的集合的逻辑与对集合中每个元素应用的行为区分开来。这样你可以重复使用同一个方法，给它不同的行为来达到不同的目的。一般这种模式需要我们提供一个行为接口和不同的行为实现类，通过将不同的行为实现类传递给处理方法来达到不同的处理行为。这样每次需要一个新的行为就要实现一个新的接口实现类，可以通过匿名累和lambda表达式来简化代码。

行为参数化类似设计模式中的策略模式。常见的应用有：Comparator排序和Runnable执行代码块。

# Lambda表达式

我们说它是函数，是因为Lambda函数不像方法那样属于某个特定的类。但和方法一样，Lambda有参数列表、函数主体、返回类型，还可能有可以抛出的异常列表。

基本语法：

(parameters) -> expression

(parameters) -> {statements;}

注意带括号的区别，一个是表达式一个是语句。

函数式接口就是只定义一个抽象方法的接口，可以在函数式接口上使用Lambda表达式。Lambda表达式的签名要和接口中抽象方法的签名相同。

@FunctionalInterface表示该接口会被设计成一个函数式接口，如果接口违背了函数式接口的定义会报错。

Java 8新增的函数式接口：

* Predicate

  ```java
  @FunctionalInterface
  public interface Predicate<T>{
    boolean test(T t);
  }
  ```

* Consumer

  ```java
  @FunctionalInterface
  public interface Consumer<T>{
    void accept(T t);
  }
  ```

* Function

  ```java
  @FunctionalInterface
  public interface Function<T,R>{
    R apply(T t);
  }
  ```

泛型只能绑定到引用类型，Java 8 也提供了针对于基本数据类型的函数式接口。

如果一个Lambda的主体是一个语句表达式，它就和一个返回void的函数描述符兼容。

Lambda表达式也允许使用自由变量，不是参数而是在外层作用域中定义的变量。Lambda可以没有限制地捕获实例变量和静态变量。但局部变量必须显式声明为final，或事实上是final。

# 方法引用

语法格式：目标引用放在分隔符::前，方法的名称放在后面。方法引用可以被看作仅仅调用特定方法的Lambda的一种快捷写法。事实上方法引用就是让你根据已有的方法实现来创建Lambda表达式。

方法引用主要有三类。

* 指向静态方法的方法引用
* 指向任意类型实例方法的方法引用。主要思想就是你在引用一个对象的方法，而这个对象本身是Lambda的一个参数。
* 指向现有对象的实例方法的方法引用。意思就是你在Lambda中调用一个已经存在的外部对象中的方法。

对于一个现有构造函数，你可以利用它的名称和关键字new来创建它的一个引用：ClassName::new。

# 默认方法

函数式接口只能包含一个抽象方法，但是通过默认方法可以为接口提供其他功能。

# 函数式数据处理

流就像是一个延迟创建的集合：只有在消费者要求的时候才会计算值。

和迭代器类似，流只能遍历一次。遍历完之后我们就说这个流已经被消费掉了。

流的处理采用内部迭代，集合的处理则是显示的外部迭代。内部迭代时项目可以透明地并行处理，或者用更优化的顺序进行处理。要是用Java过去的那种外部迭代方法，这些优化都是很困难的。

除非流水线上触发一个终端操作，否则中间操作不会执行任何处理，这是因为中间操作一般都可以合并起来在终端操作时一次性全部处理。如果流水线中存在limit类似的操作，那么整个流水线只会处理需要的数据。

常见的流操作：

* 筛选：filter，distince，limit，skip
* 映射：map，faltMap
* 查找和匹配：allMatch，anyMatch，noneMatch，findFirst，findAny
* 归约：reduce

原始类型特化流接口：IntStream、DoubleStream和LongStream，分别将流中的元素特化为int、long和double，从而避免了暗含的装箱成本。每个接口都带来了进行常用数值归约的新方法，比如对数值流求和的sum，找到最大元素的max。此外还有在必要时再把它们转换回对象流的方法。

将流转换为特化版本的常用方法是mapToInt、mapToDouble和mapToLong。这些方法和前面说的map方法的工作方式一样，只是它们返回的是一个特化流，而不是Stream<T>。

要把原始流转换成一般流可以使用boxed方法。

IntStream和LongStream的静态方法，帮助生成范围数值：range和rangeClosed。

构建流的方式：

* 集合
* 范围
* 值
* 数组
* 文件
* 函数

collect常见用法：（collect接受一个Collection接口的实现）

* 归约和汇总
* 分组
* 分区（保留了分区函数返回true或false的两套流元素列表）

Collector接口定义：

```java
public interface Collector<T,A,R>{
  //创建新容器
  Supplier<A> supplier();
  //将元素添加到新容器
  BiConsumer<A,T> accumulator();
  //转化为最终返回的容器类型
  Function<A,R> finisher();
  //合并并行处理拆分的子流
  BinaryOperator<A> combiner();
  //设置收集器的行为标识
  //UNORDERED——归约结果不受流中项目的遍历和累积顺序的影响。
  //CONCURRENT——可以并行归约流。如果收集器没有标为UNORDERED，那它仅在用于无序数据源时才可以并行归约。
  //IDENTITY_FINISH——累加器对象将会直接用作归约过程的最终结果。
  Set<Characteristics> characteristics();
}
```

对于IDENTITY_FINISH的收集操作，还有一种方法可以得到同样的结果而无需从头实现新的Collectors接口。Stream有一个重载的collect方法可以接受另外三个函数——supplier、accumulator和combiner。

# 并行流

parallel方法会将顺序流转换为并行流，sequential方法同样可以将并行流转换为顺序流。前面提到过在一个流水线操作上当遇到终端操作时前面的流操作才会执行，所以流水线中的最后一次出现的配置决定该条流水线是否并行处理。并行流处理内部使用fork/join执行，默认的线程数就是处理器个数。

分支/合并框架的目的是以递归方式将可以并行的任务拆分成更小的任务，然后将每个子任务的结果合并起来生成整体结果。它是ExecutorService接口的一个实现，它把子任务分配给线程池（称为ForkJoinPool）中的工作线程。

Spliterator接口可以用来实现并行处理的具体分割方法。

```java
public interface Spliterator<T> {
  boolean tryAdvance(Consumer<?superT>action);
  Spliterator<T> trySplit();
  long estimateSize();
  int characteristics();
}
```

Lambda表达式和匿名类中的this和super含义是不同的，在匿名类中，this代表的是类自身，但是在Lambda中，它代表的是包含类。其次，匿名类可以屏蔽包含类的变量，而Lambda表达式不能。匿名类的类型是在初始化时确定的，但是Lambda的类型取决于它的上下文。

流提供的peek方法在分析Stream流水线时，能将中间变量的值输出到日志中，是非常有用的工具。

# 默认方法

通过关键字default进行修饰的方法。可以用于在不影响现有接口实现类的基础上进行接口的拓展。默认方法也为方法的多继承提供了一种更灵活的机制。

# Optional

在类中如果将某个字段声明为Optional，该字段不支持序列化。

# CompletableFuture

一般异步计算的任务如果出现错误，抛出的异常只会被限制在当前线程，我们无法知道具体的错误原因。利用使用CompletableFuture的completeExceptionally方法可以将异常抛出。

CompletableFuture可以定制执行任务的线程池。

# java.time工具包

* LocalDate
* LocalTime
* LocalDateTime
* Instant //机器时间表示
* Duration
* Period

# 函数式编程

声明式编程强调的是做什么，命令式编程强调的是怎么做，函数式编程是声明式编程的一种实践。

在函数式编程的上下文中，一个“函数”对应于一个数学函数：它接受零个或多个参数，生成一个或多个结果，并且不会有任何副作用，你可以把它看成一个黑盒。

递归的思想更接近于函数编程，强调做什么而不是怎么做可以帮助问题的解决，多尝试使用递归来替代迭代。

科里化是一种将具备2个参数的函数f转化为使用一个参数的函数g，并且这个函数的返回值也是一个函数，它会作为新函数的一个参数。

使用函数式编程我们可以创建延时数据结构，在需要的时候通过函数生成数据。

# java 并发知识

## volatile实现

volatile是轻量级的synchronized，不会引起上下文的切换。如果一个字段被声明成volatile，Java线程内存模型确保所有线程看到这个变量的值是一致的。对声明了volatile的变量进行写操作，JVM就会向处理器发送一条Lock前缀的指令，Lock前缀指令会实现一下两种操作：

* 将当前处理器缓存行的数据写回到系统内存。
* 这个写回内存的操作会使在其他CPU里缓存了该内存地址的数据无效。这就是缓存一致性协议，每个处理器通过嗅探在总线上传播的数据来检查自己缓存的值是不是过期了，当处理器发现自己缓存行对应的内存地址被修改，就会将当前处理器的缓存行设置成无效状态，当处理器对这个数据进行修改操作的时候，会重新从系统内存中把数据读到处理器缓存里。

## synchronized的实现

JVM基于进入和退出Monitor对象来实现方法同步和代码块同步，底层由monitorenter和monitorexit指令实现。。任何对象都有一个monitor与之关联，当且一个monitor被持有后，它将处于锁定状态。线程执行到monitorenter指令时，将会尝试获取对象所对应的monitor的所有权，即尝试获得对象的锁。

synchronized支持可重入。

synchronized的锁信息存储在java对象头中：

![截屏2020-04-09 下午9.00.26](https://tva1.sinaimg.cn/large/00831rSTgy1gdntcr48vnj31q00au10f.jpg)

## 偏向锁

当一个线程访问同步块并获取锁时，会在对象头和栈帧中的锁记录里存储锁偏向的线程ID，以后该线程在进入和退出同步块时不需要进行CAS操作来加锁和解锁，只需简单地测试一下对象头的MarkWord里是否存储着指向当前线程的偏向锁。偏向锁在有其他线程尝试竞争锁时才会释放。

## 轻量级锁

线程在执行同步块之前，JVM会先在当前线程的栈桢中创建用于存储锁记录的空间，并将对象头中的MarkWord复制到锁记录中，官方称为DisplacedMarkWord。然后线程尝试使用CAS将对象头中的MarkWord替换为指向锁记录的指针。如果成功，当前线程获得锁，如果失败，表示其他线程竞争锁，当前线程便尝试使用自旋来获取锁。自旋一段时间任未获取到锁时此时当前线程会将当前所类型改为重量级锁，锁一旦升级就不会降级。

轻量级解锁时，会使用原子的CAS操作将DisplacedMarkWord替换回到对象头，如果成功，则表示没有竞争发生。如果失败，表示当前锁存在竞争，锁就会膨胀成重量级锁。

对于重量级锁来说如果存在锁竞争会直接阻塞当前线程。

## 原子操作

处理器实现原子操作是通过总线锁或者缓存锁，其中总线锁就是锁定cpu和内存之间的操作。

java通过锁和CAS实现原子操作。其中CAS存在ABA问题，循环开销大，只能保证一个变量操作的原子性。

## Java内存模型

java并发采用的是共享内存模型。

在Java中，所有实例域、静态域和数组元素都存储在堆内存中，堆内存在线程之间共享。局部变量，方法定义参数和异常处理器参数不会在线程之间共享。

在执行程序时，为了提高性能，编译器和处理器常常会对指令做重排序。这些重排序会导致多线程程序出现可见性问题，JMM通过禁止一些类型的重排序和插入内存屏障提供一致的内存可见性保障。重排序的前提是不会改变单线程的执行结果，对于存在数据依赖关系（同一处理器，同一线程之间的依赖）的语句不会进行重排序。

内存屏障保证的是操作可见性的问题。

数据通过总线在处理器和内存之间传递，一次传递称为一个总线事物。同一时间只能存在一个总线事物，总线的这种工作机制可以把所有处理器对内存的访问以串行化的方式来执行。在任意时间点，最多只能有一个处理器可以访问内存。这个特性确保了单个总线事务之中的内存读/写操作具有原子性。

JMM不保证对64位的long型和double型变量的写操作具有原子性。

volatile重排序规则：

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1gdpmg70gxej31p20cqn30.jpg" alt="截屏2020-04-11 上午10.32.41" style="zoom:50%;" />

通过一下的内存屏障插入策略来实现上述的重排序规则：

* 在每个volatile写操作的前面插入一个StoreStore屏障。
* ·在每个volatile写操作的后面插入一个StoreLoad屏障。
* ·在每个volatile读操作的后面插入一个LoadLoad屏障。·
* 在每个volatile读操作的后面插入一个LoadStore屏障。

锁的释放/获取和volatile的读写具有相同的内存语义，释放锁时会将本地内存的数据刷到主内存，获取锁时会将本地内存置为无效。锁的获取和释放还是基于volatile和CAS操作进行的，对于公平锁来说讲究先来后到，竞争锁时会判断自己是否为等待队列的队头，而非公平锁会直接CAS获取锁，成功就执行。

CAS具有和volatile相同的内存语义，是基于机器级别的原子指令实现的。

final的重排序规则：

* 在构造函数内对一个final域的写入，与随后把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序。
* 初次读一个包含final域的对象的引用，与随后初次读这个final域，这两个操作之间不能重排序。

在执行类的初始化期间，JVM会去获取一个锁。这个锁可以同步多个线程对同一个类的初始化。

## 线程通信

管道输入/输出流适用于线程间的数据传输，传输媒介为内存。

## AQS

队列同步器AbstractQueuedSynchronizer（以下简称同步器），是用来构建锁或者其他同步组件的基础框架，它使用了一个int成员变量表示同步状态，通过内置的FIFO队列来完成资源获取线程的排队工作。锁是面向使用者的，AQS是面向锁的实现者的。java提供了lock接口面向使用者，自定义的锁组件都是通过实现lock接口，实现一个内部类继承AQS并实现模版方法，通过该内部类实现同步状态的获取和释放。

下面是AQS的重要组成部分同步队列，当线程获取同步状态失败后会被加入该队列，头节点表示成功获取同步状态的线程信息，尾结点的设置会涉及到多线程的竞争，所以通过CAS进行设置。

![截屏2020-04-13 下午8.34.26](https://tva1.sinaimg.cn/large/007S8ZIlgy1gdsf2wk3sjj31ei0fgk08.jpg)

读写锁将同步状态按位切割使用，高位表示读状态，低位表示写状态。

Java 提供了LockSupport工具，该工具类中提供了用于阻塞和唤醒线程的方法。

## Condition

Condition定义了等待/通知两种类型的方法，当前线程调用这些方法时，需要提前获取到Condition对象关联的锁。当调用await()方法后，当前线程会释放锁并在此等待，而其他线程调用Condition对象的signal()方法，通知当前线程后，当前线程才从await()方法返回，并且在返回前已经获取了锁。

ConditionObject是同步器AbstractQueuedSynchronizer的内部类。Condition包含一个等待队列，其结构和AQS中的同步队列类似，但是这个等待队列的尾结点的设置不需要进行CAS操作，因为操作Condition对象之前已经获取到了锁，可以保证线程安全。

![截屏2020-04-17 下午8.05.19](https://tva1.sinaimg.cn/large/007S8ZIlgy1gdx0puq18lj31bu0cw0ym.jpg)

在Object的监视器模型上一个对象拥有一个同步队列和等待队列，而同步器拥有一个同步队列和多个等待队列。

![截屏2020-04-17 下午8.10.29](https://tva1.sinaimg.cn/large/007S8ZIlgy1gdx0v6tn5tj31ig0u0h4y.jpg)

调用Condition对象的await方法实际上就是将同步队列的首结点移动到Condition的等待队列。

调用Condition的signal()方法，将会唤醒在等待队列中的首节点，在唤醒节点之前会将节点移到同步队列中。

## Java并发容器和框架

### ConcurrentHashMap

HashTable容器使用synchronized来保证线程安全，但在线程竞争激烈的情况下HashTable的效率非常低下。ConcurrentHashMap通过分段锁有效提高并发访问的效率。

ConcurrentHashMap是由Segment数组结构和HashEntry数组结构组成。Segment是一种可重入锁，在ConcurrentHashMap里扮演锁的角色；HashEntry则用于存储键值对数据。一个ConcurrentHashMap里包含一个Segment数组。Segment的结构和HashMap类似，是一种数组和链表结构。一个Segment里包含一个HashEntry数组，每个HashEntry是一个链表结构的元素，每个Segment守护着一个HashEntry数组里的元素，当对HashEntry数组的数据进行修改时，必须首先获得与它对应的Segment锁。

segment数组的长度确保为2的N次方。get操作不需要加锁，因为和get操作相关的共享变量都是volitile类型，并且get操作不涉及到对共享变量的写。

### ConcurrentLinkedQueue

入队操作为了保证线程安全需要CAS操作更新tail结点，这样会导致入队操作性能下降。所以ConcurrentLinkedQueue设置了变量hops，当tail结点和尾结点的长度超过该变量时才会CAS更新tail结点，这样会提高入队效率，但同时也会延长定位尾结点的时间。出队操作也是这样更新head结点。

出队首先获取头节点的元素，然后判断头节点元素是否为空，如果为空，表示另外一个线程已经进行了一次出队操作将该节点的元素取走，则取下一个结点。如果不为空，则使用CAS的方式将头节点的引用设置成null，如果CAS成功，则直接返回头节点的元素，如果不成功，表示另外一个线程已经进行了一次出队操作更新了head节点，导致元素发生了变化，需要重新获取头节点。

### 阻塞队列

当阻塞队列满时会阻塞入队线程，当队列空时会阻塞出队线程。

JDK7提供了7个阻塞队列，如下：

* ArrayBlockingQueue：一个由数组结构组成的有界阻塞队列。
* LinkedBlockingQueue：一个由链表结构组成的有界阻塞队列。
* PriorityBlockingQueue：一个支持优先级排序的无界阻塞队列。
* DelayQueue：一个使用优先级队列实现的无界阻塞队列。
* SynchronousQueue：一个不存储元素的阻塞队列，一个入队操作必须等待一个出队操作。
* LinkedTransferQueue：一个由链表结构组成的无界阻塞队列。
* LinkedBlockingDeque：一个由链表结构组成的双向阻塞队列。

阻塞队列一般通过Condition实现。

### Fork/Join

工作窃取算法是指线程从其他线程的工作队列中获取任务执行，这样是为了减少因为线程空闲造成的资源闲置，当前线程任务处理结束后可以通过该机制帮助其他线程进行任务处理。为了减少多个线程处理同一队列造成的竞争关系，一般采用双端队列。

### CountDownLatch

CountDownLatch允许一个或多个线程等待其他线程完成操作。

### CyclicBarrier

让一组线程到达一个同步点时被阻塞，直到最后一个线程到达同步点时所有阻塞在同步点的线程才会继续运行。CyclicBarrier可以通过reset方法进行充值。

### Semaphore

用来控制同时访问特定资源的线程数。

### Exchanger

用于进行线程间的数据交换。它提供一个同步点，在这个同步点，两个线程可以交换彼此的数据。这两个线程通过exchange方法交换数据，如果第一个线程先执行exchange()方法，它会一直等待第二个线程也执行exchange方法，当两个线程都到达同步点时，这两个线程就可以交换数据，将本线程生产出来的数据传递给对方。

## Java线程池

线程池的主要处理流程：

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1gdy2wmknhlj31rk0u0dxe.jpg" alt="截屏2020-04-18 下午6.06.35" style="zoom:50%;" />

ThreadPoolExecutor的执行示意图：

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1gdy3457n2zj30yj0u018b.jpg" alt="截屏2020-04-18 下午6.13.50" style="zoom:50%;" />

在1，3两步中创建新线程执行任务时需要获取全局锁的，ThreadPoolExecutor的这种设计就是为了尽量避免全局锁的获取，使的在大部分情况下线程池都是在执行步骤2。

可以使用两个方法向线程池提交任务，分别为execute()和submit()方法。execute()方法用于向线程池提交不需要返回值的任务，submit()方法会返回一个Future对象用来判断任务执行情况并获取返回值。

可以通过调用线程池的shutdown或shutdownNow方法来关闭线程池。它们的原理是遍历线程池中的工作线程，然后逐个调用线程的interrupt方法来中断线程，所以无法响应中断的任务可能永远无法终止。但是它们存在一定的区别，shutdownNow首先将线程池的状态设置成STOP，然后尝试停止所有的正在执行或暂停任务的线程，并返回等待执行任务的列表，而shutdown只是将线程池的状态设置成SHUTDOWN状态，然后中断所有没有正在执行任务的线程。

## Executor

和线程池差不多。

## Java 并发课程补充

导致并发问题的源头：

* 缓存可见性问题
* 线程切换的原子性问题
* 编译器优化带来的有序性问题

java内存模型规范了JVM如何提供按需禁用缓存和编译优化的方法。

happens-before规则表达的是：前面一个操作的结果对后续操作是可见的。常见的六项规则：

* 程序的顺序性规则（编译器的重排序的优化是基于在单线程下不会影响程序的执行结果）
* volatile变量规则
* 传递性
* 管程中锁的规则
* 线程start规则
* 线程join规则

上锁是必须确保是同一个锁对象，有些不可变对象重新赋值后就会变成另一个对象。

解决死锁的常用套路：

* 破坏占用且等待条件（一次申请所有资源）
* 破坏不可抢占条件
* 破坏循环等待条件

管程：管理共享变量以及对共享变量的操作过程，让他们支持并发。java中的管程示意图：

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1ghcpz7cnz0j30i60ha0un.jpg" alt="截屏2020-08-02 下午8.03.44" style="zoom:50%;" />

线程的五态模型：

<img src="/Users/chenjie/Library/Application Support/typora-user-images/截屏2020-08-02 下午8.06.46.png" alt="截屏2020-08-02 下午8.06.46" style="zoom:50%;" />

初始状态只是编程语言层面的，这个时候在操作系统层面真正的线程还没有被创建。可运行状态是指操作系统已经创建了该线程，可以被分配CPU执行。java中讲可运行状态和运行状态归为一体，因为线程的调度发生在操作系层面。

stop方法会直接杀死线程，interrupt方法仅仅是通知线程。

抛出异常时会自动清除中断标示。

在并发编程领域，提升性能本质上就是提升硬件的利用率，再具体点来说，就是提升 I/O 的利用率和 CPU 的利用率。

ReadWriteLock 不支持锁升级，支持锁降级。

StampedLock提供三种模式锁：写锁，悲观读锁和乐观读，乐观读是不需要加锁的。

java并发容器：

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1ghkkw6j44kj30vu0aowhz.jpg" alt="截屏2020-08-09 下午3.12.24" style="zoom:50%;" />

CopyOnWriteArrayList：执行写操作时会讲内部维护的数组复制一份，在新的数组上进行写操作，操作完成后将数组指针替换为新数组。在这期间读写会存在一个不一致的时间窗口。该容器的迭代器只读。

ConcurrentHashMap 的 key 是无序的，而 ConcurrentSkipListMap 的 key 是有序的。

java线程池是一种生产者-消费者模式，工作线程作为消费者不断从队列中获取任务，工作线程的个数就是我们说的线程数。

Future 接口有 5 个方法，它们分别是取消任务的方法 cancel()、判断任务是否已取消的方法 isCancelled()、判断任务是否已结束的方法 isDone()以及2 个获得任务执行结果的 get() 和 get(timeout, unit)。FutureTask 实现了 Runnable 和 Future 接口，由于实现了 Runnable 接口，所以可以将 FutureTask 对象作为任务提交给 ThreadPoolExecutor 去执行，也可以直接被 Thread 执行；又因为实现了 Future 接口，所以也能用来获得任务的执行结果。

## CompletableFuture

创建CompletableFuture对象（实现了Future接口）：（默认使用公共的 ForkJoinPool 线程池）

* static CompletableFuture<Void>   runAsync(Runnable runnable) //无返回值
* static <U> CompletableFuture<U>   supplyAsync(Supplier<U> supplier) //有返回值
* static CompletableFuture<Void>   runAsync(Runnable runnable, Executor executor)
* static <U> CompletableFuture<U>   supplyAsync(Supplier<U> supplier, Executor executor)  

CompletableFuture 还能规定任务的执行时序关系，具体应用自己google。

## CompletionService

CompletionService 可以批量执行异步任务，它会将异步任务的执行结果放在内部维护的一个阻塞队列中。

# 并发设计模式

## 享元模式

享元模式本质上其实就是一个对象池，利用享元模式创建对象的逻辑也很简单：创建之前，首先去对象池里看看是不是存在；如果已经存在，就利用对象池里的对象；如果不存在，就会新创建一个对象，并且把这个新创建出来的对象放进对象池里。Java 语言里面 Long、Integer、Short、Byte 等这些基本数据类型的包装类都用到了享元模式。所以这些基本数据类型不适合作为锁对象，会导致看上去私有的锁其实是共有的。

## Immutability模式

## 线程本地存储模式

ThreadLocal内部实现：

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1ghsgvblajej30ak0a2myi.jpg" alt="截屏2020-08-16 上午10.57.40" style="zoom:50%;" />

通过 ThreadLocal 创建的线程变量，其子线程是无法继承的，Java 提供了 InheritableThreadLocal 来支持这种特性。

在线程池中使用 ThreadLocal 可能导致内存泄露呢，原因就是线程池中线程的存活时间太长，这就意味着 Thread 持有的 ThreadLocalMap 一直都不会被回收，再加上 ThreadLocalMap 中的 Entry 对 ThreadLocal 是弱引用，所以只要 ThreadLocal 结束了自己的生命周期是可以被回收掉的。但是 Entry 中的 Value 却是被 Entry 强引用的，所以即便 Value 的生命周期结束了，Value 也是无法被回收的，从而导致内存泄露。

## Guarded Suspension 模式

## Balking 模式

## Thread-Per-Message 模式

## Worker Thread 模式

## 二阶段终止模式

首先需要利用中断将线程状态转换为运行时。

# Java 小记

* Java变量的初始化顺序为：静态变量或静态语句块–>实例变量或初始化语句块–>构造方法–>@Autowired







# Effective Java

## 创建和销毁对象

* 静态方法代替构造器

* 遇到多个构造器参数时考虑构建器

  构建器将对象的构建放在单独的Builder方法中，当调用build方法时才实例化对象。

* 用私有构造器或者枚举类型强化Singleton属性

  对象序列化会破坏Singleton语义，在反序列化的时候会创建一个新的实例。为了保证单例所有实例域都必须为transient类型，并提供readResolve方法。但是枚举类型比较特殊，拥有自己独特的序列化机制，忽略所有定制化序列化机制。在序列化的时候Java仅仅是将枚举对象的name属性输出到结果中，反序列化的时候则是通过java.lang.Enum的valueOf方法来根据名字查找枚举对象。

* 通过私有构造器强化不可实例化的能力

*  优先考虑依赖注入来引用资源

* 避免创建不必要的对象（缓存，自动装箱）

* 58清楚过期的对象引用

* 避免使用终结方法和清除方法

* try-with-resources优于try-finally

## 所有对象通用方法

* 覆盖equals时遵守通用约定

  默认的equals方法比较两个对象地址是否相等

* 覆盖equals时总要覆盖hashCode

* 始终要覆盖toString

* 考虑实现Comparable接口

## 类和接口

# JVM调优

