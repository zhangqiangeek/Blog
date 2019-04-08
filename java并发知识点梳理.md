---
   title: java并发知识点整理
   date: 2018-10-1 8:19:00
   tags: java
---

![Java并发知识图谱](http://pic.evilhex.com/2019-04-08-Java并发知识图谱.png)

### CPU多级缓存
- EMSI
- 乱序执行优化

### java内存模型
- 同步八种操作与规则

### 并发编程与线程安全
- 原子性
- 可见性
- 有序性
<!-- more -->
#### 保证原子性的手段：
1、Atomic包

AtomicXXX:CAS、Unsafe.compareAndSwapInt

AtomicLong、LongAdder

AtomicReference、AtomicReferenceFieldUpdater

AtomicStampReference :CAS的ABA问题

2、锁

Synchronized:JVM

Lock:依赖CPU的指令

#### 保证可见性的手段

> 导致共享变量在线程间不可见的原因： 线程交叉执行；重排序结合线程交叉执行；共享变量更新后的值没有在工作内存和主内存及时更新；

JMM关于synchronized的两条规定：

- 线程解锁前，必须把共享变量的最新值刷新到主内存中；
- 线程加锁时，将清空工作内存中共享变量的值，从而使用共享变量时需要从主内存中重新读取最新的值。

volatile通过加入内存屏障和禁止指令重排序优化来实现：

- 对volatile变量写操作时，会在写操作后加入一条store屏障指令，将本地内存中的共享变量刷新到主内存
- 对volatile变量读操作时，会在读操作前加入一条load屏障指令，从主内存中读取共享变量

> volatile适合状态标记量；不适合修饰count++这种操作；
另一个使用场景：double check

#### 保证有序性的手段：

- java内存模型中，允许编译器和处理器对指令进行重排序。重排序的过程不会影响到单线程的执行，却会影响到多线程并发执行的正确性。
- volatile synchronized Lock

##### 有序性 happens-before原则
1. 程序次序规则：一个线程内，按照代码顺序，书写在前边的操作先行于书写在后边的操作；
2. 锁定规则：一个unlock操作先行发生于后面对同一个锁的lock操作；
3. volatile变量规则：对一个变量的写操作先行发生于后面对这个变量的读操作；
4. 传递规则：如果操作A先行发生于B，而操作B又先行发生于C，则可以得出操作A先行发生于C；
5. 线程启动规则：Thread对象的start()方法先行发生于此线程的每一个动作；
6. 线程中断规则：对线程interrupt()方法的调用先行发生于被中断线程的代码检测到中断事件的发生；
7. 线程终结规则：线程中所有的操作都先行发生于线程的终止检测，我们可以通过Thread.join()方法结束、Thrad.isAlive()方法的返回值手段检测到线程已经终止执行；
8. 线程终结规则：一个线程的初始化完成先行发生于他finalize（）方法的开始。

---

### 发布对象

发布对象：使一个对象能够被当前范围之外的代码使用。

对象逸出：一种错误的发布。当一个对象还没有构造完成时，就被其他线程所见。

#### 安全发布对象

- 在静态初始话函数中初始化一个对象的引用
- 将对象的引用保存到volatile类型或者AtomicReference对象中
- 将对象的引用保存到某个正确构造对象的final类型域中
- 将对象的引用保存到一个由锁保护的域中

#### 不可变对象

不可变对象满足的条件：

- 对象创建以后，其状态不能修改
- 对象所有的域都是final类型
- 对象是正确创建的（在对象创建期间，this引用没有逸出）

final关键字：类，变量，方法

- 修饰类：不能被继承
- 修饰方法：锁定方法不被继承类修改；效率
- 修饰变量：基本数据类型变量；引用类型变量 

Collections.unmodifiableXXX:Collection,List,Set,Map...

Guava:ImmutableXXX:Collection,List,Set,Map...

### 线程封闭

Ad-hoc 线程封闭：程序控制实现，最糟糕，忽略

堆栈封闭：局部变量，无并发问题

ThreadLocal线程封闭：特别好的封闭方法

### 线程不安全的类与写法

线程不安全->线程安全

StringBuilder -> StringBuffer

SimpleDateFormat -> JodaTime

ArrayList,HashSet,HashMap等collections

### 同步容器

ArrayList -> Vector,Stack

HashMap -> HashTable(key value不能为null)

Collections.synchronizedXXX(List、Set、Map)

同步容器不一定能够做到完全的线程安全，性能也不是特别好。

### 并发容器 J.U.C

ArrayList -> CopyOnWriteArrayList(读多写少的场景)

HashSet、TreeSet -> CopyOnWriteArraySet、 ConcurrentSkipListSet(自然排序) 

HashMap、TreeMap -> ConcurrentHashMap、ConcurrentSkipListMap

> J.U.C 包括五部分：tools,locks,atomic,collections,executor

### 安全共享对象策略-总结
- 线程限制：一个被线程限制的对象，由线程独占，并且只能被占有它的线程修改
- 共享只读：一个共享只读的对象，在没有额外同步的情况下，可以被多个线程并发访问，但是任何线程都不能修改它
- 线程安全对象：一个线程安全的对象或者容器，在内部通过同步机制来保证线程安全，所以其他线程无需额外的同步就可以通过公共接口随意访问它
- 被守护对象：被守护对象只能通过获取特定的锁来访问


---

### AQS

- 使用Node实现FIFO队列，可以用于构建锁或者其他同步装置的基础框架
- 利用int类型表示状态
- 使用方法是继承
- 子类通过继承并通过实现它的方法管理其状态（acquire和release）的方法操纵状态
- 可以同时实现排他锁和共享锁模式（独占、共享）

#### AQS同步组件

- CountDownLatch
- Semaphore
- CyclicBarrier
- ReentrantLock
- Condition
- FutureTask

##### CountDownLatch
##### Semaphore
做并发访问控制
##### CyclicBarrier
处理更复杂的业务场景

##### ReentrantLock和锁

- 可重入性
- 锁的实现
- 性能的区别
- 功能区别

ReentrantLock的独有功能

- 可指定是公平锁还是非公平锁：ReentrantLock可以指定是公平的还是非公平的，synchronized只能是公平的；
- 提供了一个Condition类，可以分组唤醒需要唤醒的线程
- 提供能够中断等待锁的线程的机制，lock.lockInterruptibly()

StampedLock
看源码，有的地方性能上有很大的提升

> juc中的锁还是很重要的，多看几遍要掌握

### FutureTask

- Callable和Runnable接口对比
- Future接口
- FutureTask类

### Fork/Join框架

java7提供并行执行任务的框架；
采用工作窃取算法；
线程和队列一一对应；
双端队列，从尾部窃取任务；

### BlockingQueue
- ArrayBlockingQueue 有界
- DelayQueue 定时关闭连接 超时处理等场景
- LinkedBlockingQueue 无边界
- PriorityBlockingQueue 
- SynchronousQueue 

### 线程池

#### new Thread 弊端
- 每次new Thread新建对象，性能差；
- 线程缺乏统一管理，可能无限制的创建贤臣个，相互竞争，有可能占用过多的资源导致OOM；
- 缺少更多的功能，如更多执行、定期执行、线程中断

#### 线程池的好处
- 重用存在的线程，减少对象的创建、死亡的开销，性能佳
- 可有效的控制最大并发线程数，提高系统资源利用率，同时可以避免过多资源竞争，避免阻塞
- 提供定时执行、定期执行，并发数控制等功能

#### ThreadPoolExecutor
- corePoolSize:核心线程数量
- maximumPoolSize:线程最大线程数
- workQueue：阻塞队列，存储等待执行的任务，很重要，会对线程池运行过程产生重大影响
- keepAliveTime:线程没有任务执行时最多保持多久时间终止
- unit：keepAliveTime的时间单位
- threadFactory:线程工厂，用来创建线程
- rejectHandler：当拒绝处理任务时的策略

执行方法

- execute(): 提交任务，交给线程池执行
- submit():提交任务，能够返回执行结果
- shutdown():关闭线程池，等待任务都执行完
- shutdownNow():关闭线程池，不等待任务执行完

监控方法

- getTaskCount(): 线程池已执行和未执行的任务总数
- getCompletedTaskCount():已完成的任务数量
- getPoolSize():线程池当前的线程数量
- getActiveCount():当前线程池中正在执行的线程数量

### 死锁

必要条件
- 互斥条件
- 请求和保持条件
- 不剥夺条件
- 环路等待条件

### 多线程并发最佳实践
- 使用本地变量
- 使用不可变类
- 最小化锁的作用域范围：S=1/(1-a+a/n)
- 使用线程池的Executor，不单独创建线程
- 宁可使用同步也不使用线程的wait和notify
- 使用BlockingQueue实现生产-消费模式
- 使用并发集合而不是加了锁的同步集合
- 使用Semaphore创建有界的访问
- 宁可使用同步代码块，也不使用同步的方法
- 避免使用静态变量（一般配合final）


## Spring与线程安全

- Spring bean：singleton、protetype
- 无状态对象（自身没有状态的对象）

## 高并发处理思路与手段

### 扩容

- 垂直扩容：提高系统部件能力
- 水平扩容：增加更多系统成员来实现

扩容-数据库

- 读操作扩展：memcache、redis、cdn等缓存
- 写操作扩展：Cassandra、Hbase等

### 缓存

[缓存那些事](https://tech.meituan.com/cache_about.html)

缓存特征
- 命中率：命中数/（命中数+没有命中数）
- 最大元素（空间）
- 清空策略：FIFO，LFU，LRU，过期时间，随机等

缓存命中率的影响因素

- 业务场景和业务需求
- 缓存的设计（粒度和策略）
- 缓存的容量和基础设施

缓存的分类和应用场景

- 本地缓存：缓存编程（成员变量，局部变量，静态变量），guava cache
- 分布式缓存：memcache，redis

高并发场景下缓存常见问题

- 缓存一致性
- 缓存并发问题
- 缓存穿透问题
- 缓存的雪崩现象：限流 降低 熔断 多级缓存


消息队列

消息队列特性

- 业务无关
- FIFO：先投递先到达
- 容灾：节点的动态增删和消息的持久化
- 性能：吞吐量提升，系统内部通信效率提高

为什么需要消息队列

- 生产和消费的速度或稳定性等因素不一致
- 业务解耦
- 最终一致性
- 广播
- 错峰与流控

例如：kafka RabbitMQ RocketMQ

### 应用拆分

- 业务优先
- 循序渐进
- 兼顾技术：重构、分层
- 可靠测试

思考：
- 应用之间的通信：RPC（dubbo等）、消息队列
- 应用之间的数据库设计：每个应用都有独立的数据库
- 避免实务操作跨应用

Dubbo和Spring cloud

### 应用限流

- 计数器法
- 滑动窗口
- 漏桶算法 （Leaky Bucket）
- 令牌桶算法 （Token Bucket）

### 服务降级与服务熔断

服务降级
服务熔断

服务降级分类
- 自动降级：超时、失败次数、故障、限流
- 人工降级：秒杀、双11大促等

### 数据库切库、分库、分表

##### 数据库瓶颈
- 单个库数据量太大（1T~2T）:多个库
- 单个数据库服务器压力过大、读写瓶颈：多个库
- 单个表数据量过大：分表


#### 数据库切库

- 切库的基础及实际应用：读写分离
- 自定义注解完成数据库切库：代码实现

#### 数据库分表
- 什么时间考虑分表？
- 横向分表与纵向分表
- 数据库分表：mybatis分表插件 shardbatis2.0

#### 高可用的一些手段

- 任务调度系统分布式：elastic-job+zookeeper
- 主备切换：apache curator +zookeeper 分布式锁实现
- 监控报警机制