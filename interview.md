## Redis

### 缓存策略有哪些

本地缓存、分布式缓存和多级缓存

在进程的内存中进行的缓存，比如 JVM 堆中，可以使用 LRUMap 来实现。

实际的业务中一般采用多级缓存，本地缓存只保存访问频率最高的部分热点数据，其他的热点数据放在分布式缓存中。

### Redis 的淘汰策略有哪些

不管是本地缓存还是分布式缓存，为了保证较高的性能，都是使用内存来保存数据，由于成本和内存限制，当存储的数据超过缓存容量时，需要对缓存的数据进行剔除。

剔除策略有以下六种

- 在存在过期时间的集合中，根据 LRU，回收最近最少使用的键。
- 在存在过期时间的集合中，回收剩余存活时间最少的 key。
- 在存在过期时间的集合中，随机回收 key。
- 在所有数据集合中，根据 LRU，回收最近最少使用的key。
- 在所有数据集合中，随机回收 key。
- 不回收，直接返回错误。

### 特点

单线程

按请求先后顺序，放在队列里，先后执行。

- 性能优秀，数据在内存中，读写速度非常快，支持 10w QPS。
- 单进程单线程，是线程安全的，采用IO多路复用机制。
- 丰富的数据类型，支持 String、hash、list、set、sorted set
- 支持数据持久化（RDB 和 AOF)
- 可以作为分布式锁 setnx
- 可以将 list 作为消息中间件来使用，lpush rpop

### 模式

- 单机模式
- 集群模式
- 主从模式

主从模式，跟提到的数据持久化的 RDB 和 AOF 有密切的关系。

单机 QPS 是有上限的，而且 Redis 的特点就是必须支持读高并发，但是一台机器又读有写压力比较大，所以就用 master 机器去写，数据同步给 slave 机器，让他们负责读，这样子就能分发掉大量的请求，而且扩容的时候还阔以轻松实现水平扩容。

当你启动一台 salve 的时候，他会发送一个 psync 命令给 master，如果是这个 slave 第一次连接到 master，他会触发一个全量复制。master 就会启动一个线程，生成 RDB 快照，还会把新的写请求都缓存再内存中，RDB 文件生成后，master 会将这个 RDB 发送给 slave的，slave 拿到之后做的第一件事情就是写进本地的磁盘，然后加载进入内存，然后 master 会把内存里面缓存的那些新命令都发送给 slave。之后的数据，master 会通过 AOF，把日志增量同步给 slave。







userid = 

Userid -> point 

select * from point_table where uid = xxx;

### 缓存雪崩

当某一时刻发生大规模的缓存失效的情况，比如你的缓存服务宕机了，会有大量的请求进来直接打到DB上面。结果就是DB 称不住，挂掉。或者过期时间集体到期。

解决方案：

使用集群缓存，保证缓存服务的高可用。

使用 Hystrix进行限流 & 降级，避免 mysql 被打死。

### 缓存穿透

缓存穿透是指用户查询数据，在数据库没有，自然在缓存中也不会有。这样就导致用户查询的时候，在缓存中找不到，每次都要去数据库再查询一遍，然后返回空（相当于进行了两次无用的查询）。

解决方案：

- **布隆过滤器**,将所有可能存在的数据哈希到一个足够大的bitmap中，一个一定不存在的数据会被这个bitmap拦截掉，从而避免了对底层存储系统的查询压力。
- 返回为空也存进去，过期时间放短一点。

### 缓存击穿

大量请求集中在某个 key 上，当 key 失效，就会造成大量请求落在 db 上。



### 分布式锁

#### SETNX

将 key 的值设为 value，当且仅当 key 不存在。

若给定的 key 已经存在，则 SETNX 不做任何动作。

SETNX 是 [SET if Not exists] (如果不存在，则 SET) 的简称。

```java
    @RequestMapping("/deduct_stock")
    public String buy() {
        String lockKey = "lockKey";
        try {
            Boolean result = stringRedisTemplate.opsForValue().setIfAbsent(lockKey, "zhuge", 10, TimeUnit.SECONDS);
            if (!result) {
                return "error_code";
            }

            int stock = Integer.parseInt(stringRedisTemplate.opsForValue().get("stock"));
            if (stock > 0) {
                int realStock = stock - 1;
                stringRedisTemplate.opsForValue().set("stock", realStock + "");
                System.out.println("扣减成功，剩余库存：" + realStock);
            } else {
                System.out.println("扣减失败，库存不足");
            }
        } finally {
            stringRedisTemplate.delete(lockKey);
        }
        return "end";
    }
```

谁通过 setnx 设置成功了，就拿到锁了。

要用 try - catch，不然在执行过程中出了问题，锁没有释放，就出现死锁了。

即使加了 try - catch，也可能会有问题，因为可能在拿到锁，但是没有释放锁的时候，被强行 kill -9，那么锁也就没有释放的可能了，仍然会造成死锁。



写到下面一步仍然是有问题的

```java
Boolean result = stringRedisTemplate.opsForValue().setIfAbsent(lockKey, "zhuge", 10, TimeUnit.SECONDS);
```

因为在后面的代码里，执行时间可能会超过过期时间，导致锁失效。

进一步，可能会存在某种情况，第一个线程释放了第二个线程加的锁。具体分析见（https://www.bilibili.com/video/BV1ga4y147Hr?p=3）

所以可以再加一个防范措施，key 不变，value 是 线程 id，就可以解决问题了。

#### redisson

三行代码解决分布式锁的问题

```java
@Controller
public class SetnxTest {
		
    private Redisson redisson; // 1. 定义 Redisson

    private StringRedisTemplate stringRedisTemplate;

    @RequestMapping("/deduct_stock")
    public String buy() {
        String lockKey = "lockKey";
        RLock redissonLock = redisson.getLock(lockKey);
        try {
            redissonLock.lock(); // 2. 获取锁
            int stock = Integer.parseInt(stringRedisTemplate.opsForValue().get("stock"));
            if (stock > 0) {
                int realStock = stock - 1;
                stringRedisTemplate.opsForValue().set("stock", realStock + "");
                System.out.println("扣减成功，剩余库存：" + realStock);
            } else {
                System.out.println("扣减失败，库存不足");
            }
        } finally {
            redissonLock.unlock(); // 3. 释放锁
        }
        return "end";
    }

}
```

### 五种数据类型

 Redis 内部使用一个 redisObject 对象来表示所有的 key 和 value。

redisObject 主要包含数据类型（string list hash set zset）和 编码方式（raw int ht zipmap linkedlist ziplist intset）、数据指针(ptr)、虚拟内存等信息

比如数据类型为 string，encoding 就可以为 raw 或者 int。

1. String 是 Redis 最进本的类型，String 是二进制安全的，意思是 Redis 的 String 类型可以包含任何数据，比如 jpg 图片或者 序列化的对象。String 类型的值最大能存储 512 M。

2. Hash 是一个（key-value）的集合。Redis 的Hash是一个 String 的 Key 和 Value 的映射表，Hash 特别适合存储对象。常用命令是 hget，hset，hgetall 等。

3. List 列表是简单的字符串列表，按照插入顺序排序。可以添加一个元素到列表的头部（左边）或者尾部（右边）常用命令：lpush、rpush、lpop、rpop、lrange（获取列表片段）等。

   应用场景：List 应用场景非常多，也是Redis最重要的数据结构之一，比如 Twitter 的关注列表，粉丝列表都可以用 List 结构来实现。

   Redis List



## Tomcat



## Mysql



## 操作系统



### 进程和线程的区别

**进程是系统进行资源分配和调度的一个独立单位**

**线程**是进程的一个实体，**是CPU调度和分派的基本单位**

**进程和线程的关系**

- 一个线程只能属于一个进程，而一个进程可以有多个线程，但至少有一个线程。**线程是操作系统可识别的最小执行和调度单位**
- **资源分配给进程，同一进程的所有线程共享该进程的所有资源**。 **同一进程中的多个线程共享代码段(代码和常量)，数据段(全局变量和静态变量)，扩展段(堆存储)**。但是每个线程拥有自己的栈段，栈段又叫运行时段，用来存放所有局部变量和临时变量
- **处理机分给线程，即真正在处理机上运行的是线程**
- 线程在执行过程中，需要协作同步。**不同进程的线程间要利用消息通信的办法实现同步**

**进程与线程的区别**

- 进程**有自己的独立地址空间，线程没有**
- 进程是**资源分配的最小单位，线程是CPU调度的最小单位**
- 进程和线程通信方式不同(线程之间的通信比较方便。同一进程下的线程共享数据（比如全局变量，静态变量），通过这些数据来通信不仅快捷而且方便，**当然如何处理好这些访问的同步与互斥正是编写多线程程序的难点。**而进程之间的通信只能通过[进程通信](https://link.zhihu.com/?target=http%3A//baike.baidu.com/view/549640.htm)的方式进行。)
- 进程上下文切换开销大，线程开销小
- 一个进程挂掉了不会影响其他进程，而线程挂掉了会影响其他线程
- 对进程操作一般开销都比较大，对线程开销就小了

**为什么进程上下文切换比线程上下文切换代价高？**

### 线程同步的方式有哪些？

- 互斥量：采用互斥对象机制，只有拥有互斥对象的线程才有访问公共资源的权限。因为互斥对象只有一个，所以可以保证公共资源不会被多个线程同时访问。
- 信号量：它允许同一时刻多个线程访问同一资源，但是需要控制同一时刻访问此资源的最大线程数量。
- 事件（信号）：通过通知操作的方式来保持多线程同步，还可以方便的实现多线程优先级的比较操作。

### 进程间通信的方式有哪些

- 管道：半双工，只允许有亲缘关系的进程间通信
- 有名管道：半双工，允许无亲缘关系的进程间进行通信。
- 系统 IPC（消息队列、信号量、共享内存）
- Socket 通信

### 缓冲区溢出攻击

![image-20200702001803565](/Users/xmly/Documents/task/notes/imgs/image-20200702001803565.png)

按照冯·诺依曼存储程序原理，程序代码是作为二进制数据存储在内存的，同样程序的数据也在内存中，因此直接从内存的二进制形式上是无法区分哪些是数据哪些是代码的，这也为缓冲区溢出攻击提供了可能。

上图是进程地址空间分布的简单表示。代码存储了用户程序的所有可执行代码，在程序正常执行的情况下，程序计数器（PC指针）只会在代码段和操作系统地址空间（内核态）内寻址。数据段存储了用户的静态变量和全局变量。栈空间存储了用户程序的函数栈帧（包括参数、局部数据等），实现函数调用机制，其他的内存区域都可能作为缓冲区，因此缓冲区溢出的位置可能在数据段，也可能在堆、栈段。如果程序的代码有软件漏洞，恶意程序会“教唆”程序计数器从上述缓冲区内取指，执行恶意程序提供的数据代码！



缓冲区溢出是指当计算机向缓冲区填充数据时超出了缓冲区本身的容量，溢出的数据覆盖在合法数据上。

危害有以下两点：

- 程序崩溃，导致拒绝额服务
- 跳转并且执行一段恶意代码



### 什么是死锁？死锁产生的条件？

在两个或者多个并发进程中，如果每个进程持有某种资源而又等待其它进程释放它或它们现在保持着的资源，在未改变这种状态之前都不能向前推进，称这一组进程产生了死锁。通俗的讲就是两个或多个进程无限期的阻塞、相互等待的一种状态。

死锁产生的四个条件（有一个条件不成立，则不会产生死锁）

- 互斥条件：一个资源一次只能被一个进程使用
- 请求与保持条件：一个进程因请求资源而阻塞时，对已获得资源保持不放
- 不剥夺条件：进程获得的资源，在未完全使用完之前，不能强行剥夺
- 循环等待条件：若干进程之间形成一种头尾相接的环形等待资源关系

### 内存管理

![image-20200701234721439](/Users/xmly/Documents/task/notes/imgs/program-img.png)

真实的物理内存可能是上面这样子的，从 32KB 处作为开始，48KB作为结束（地址空间）。那 32/48 可不可以动态设置呢，只要在 CPU 上整两个寄存器，Base 和 Bounds 就可以了，Base 指明从哪里开始，Bounds 指定哪里是边界。因此，真是物理地址和虚拟地址之间的关系是：

physical address = virtual address + Base



## Server

### Reactor 线程模型

- 单线程模式
- 多线程模式（单Reactor）
- 多线程模式（多 Reactor)

#### 单线模式

单线程模式是最简单的 Reactor 模式。Reactor 线程是个多面手，负责多路分离套接字，Accept 新连接，并分派请求到处理器链中。该模型适用于处理器链



### muduo 网络库分析

所谓Reactor模式，是有一个循环的过程，监听对应事件是否触发，触发时调用对应的callback进行处理。

















## 设计题



