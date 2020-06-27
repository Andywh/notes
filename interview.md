## Redis

### 特点

单线程

按请求先后顺序，放在队列里，先后执行。

### 模式

- 集群模式
- 哨兵模式
- 主从模式

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



## Tomcat



## Mysql



## 设计题



