### Redis参考手册
http://doc.redisfans.com/
### Redis持久化方式以及不同方式的优缺点
由于Redis的数据都存放在内存中，如果没有配置持久化，redis重启后数据就会丢失，于是需要开启redis的持久化功能，将数据保存在磁盘上，当redis重启后，可以从磁盘中恢复数据。
***
* RDB方式
通过使用bgsave命令 进行数据快照
优点：对redis对外提供的读写服务，影响非常小，可以让redis保持高性能，并且相对于AOF持久化机制来说，直接基于RDB数据文件来重启和恢复redis进程，更加快速。
缺点：因为是周期新的，那么当系统一旦在定时持久化之前出现宕机现象，此前没有来得及写入磁盘的数据都将丢失。
* AOF方式
以日志的形式记录服务器所处理的每一个写、删等操作，查询操作不会记录，以文本的方式记录，可以打开文件看到想i的操作记录。在恢复数据时，可以直接读取该文件进行数据恢复。
优点：保存的数据额更加完整，能够更好的通过文件完成数据的重建
缺点：由于AOF一般每秒都在存储，可能对于相同数量的数据集而言，AOF文件统长要大于RDB文件，RDB文件在恢复大数据集时的数据比AOF的恢复速度要快
### 什么是缓存穿透？如何避免？
缓存穿透是指查询一个不存在的数据，由于缓存未命中时要从数据库查询，大量查询不存在的key会对数据库造成很大的压力，这就是缓存穿透
***
解决方法：
* 布隆过滤 对所有可能查询的参数以hash存储，在控制层先进性校验，不符合则丢弃，还有就是采用布隆过滤器，将的数据哈希到一个足够大bitmap里，那么不存在的数据会被bitmap拦截掉，从而避免了对底层存储系统的查询压力
* 缓存空对象，将null(查不到或者系统异常)都变成一个值，然后进行缓存，后设置较短的过期时间，让其自动剔除
### 什么是缓存雪崩？如何避免？
缓存雪崩是指缓存集中在一段时间内失效，发生大量的缓存穿透，导致所有查询都落到数据库上，就是缓存雪崩
***
解决办法：
* 永不过期，将某些热点数据设置为永不过期
* 加锁排队，限流算法(计数法，滑动窗口算法，令牌桶，漏桶等)
*  数据预热，设置缓存reload机制，预先去更新缓存。在发生大并发访问之前手动触发加载并缓存不同的key，设置不同的过期时间，让缓存失效的时间点击尽量均匀
* 做多级缓存e或者memcached，先查redis再查memcached
### 为什么Redis 单线程却能支撑高并发？
* 纯内存操作
* 核心是基于非阻塞的IO多路复用机制 在 I/O 多路复用模型中，最重要的函数调用就是 select，该方法的能够同时监控多个文件描述符的可读可写情况，当其中的某些文件描述符可读或者可写时，select 方法就会返回可读以及可写的文件描述符个数
* 单线程避免了多线程的频繁上下文切换问题
### Redis淘汰策略
通过在Redis安装目录下面的redis.conf配置文件中添加以下配置设置内存大小
//设置Redis最大占用内存大小为100M
maxmemory 100mb
配置的内存用完的时。就会通过策略来处理内存用完的情况
***
* volatile-lru：从设置过期时间的数据集中挑选出最近最少使用的数据淘汰，没有设置过期时间的key不会被淘汰，这样就可以在增加内存空间的同时保证需要持久化的数据不会丢失
* volatile-ttl：(Time To Live)生存时间算法，在设置了过期时间的key中，根据key的过期时间进行淘汰，越早过期的越优先被淘汰
* volatile-random：从设置过期时间的数据集中任意选择数据淘汰。当内存达到限制无法写入非过期时间的数据集时，可以通过该淘汰策略在主键空间中随机移除某个key
* volatile-lfu (redis5.0新增) ：从已设置过期时间的数据集挑选使用频率最低的数据淘汰
* allkeys-lru：从数据集中挑选最近最少使用的数据淘汰，该策略要淘汰的key面向的是全体key集合，而非过期的key集合
* allkeys-lfu (redis5.0新增) ：从数据集中挑选使用频率最低的数据淘汰
* allkeys-random：从数据集中选择任意数据淘汰
* no-enviction：禁止驱逐数据，也就是当内存不足以容纳新的数据时对于写请求不再提供服务，直接返回错误，请求可以继续进行，线上任务也不能持续进行，采用no-enviction策略可以保证数据不被丢失，这也是系统默认的一种淘汰策略
### Redis如何实现分布式锁
* 获取锁过程很简单，客户端向Redis发送命令：
```java
SET resource_name my_random_value NX PX 30000
my_random_value是由客户端生成的一个随机字符串，它要保证在足够长的一段时间内在所有客户端的所有获取锁的请求中都是唯一的。NX表示只有当resource_name对应的key值不存在的时候才能SET成功。这保证了只有第一个请求的客户端才能获得锁，而其它客户端在锁被释放之前都无法获得锁。PX 30000表示这个锁有一个30秒的自动过期时间。
```
* 释放锁
```java
if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1])
else
    return 0
end之前获取锁的时候生成的my_random_value 作为参数传到Lua脚本里面，作为：ARGV[1],而 resource_name 作为KEYS[1]。Lua脚本可以保证操作的原子性。
```
完整代码：
```java
@Slf4j
public class RedisDistributedLock {
 private static final String LOCK_SUCCESS = "OK";
 private static final Long RELEASE_SUCCESS = 1L;
 private static final String SET_IF_NOT_EXIST = "NX";
 private static final String SET_WITH_EXPIRE_TIME = "PX";
 // 锁的超时时间
 private static int EXPIRE_TIME = 5 * 1000;
 // 锁等待时间
 private static int WAIT_TIME = 1 * 1000;
 private Jedis jedis;
 private String key;
 public RedisDistributedLock(Jedis jedis, String key) {
  this.jedis = jedis;
  this.key = key;
 }
 // 不断尝试加锁
 public String lock() {
  try {
   // 超过等待时间，加锁失败
   long waitEnd = System.currentTimeMillis() + WAIT_TIME;
   String value = UUID.randomUUID().toString();
   while (System.currentTimeMillis() < waitEnd) {
    String result = jedis.set(key, value, SET_IF_NOT_EXIST, SET_WITH_EXPIRE_TIME, EXPIRE_TIME);
    if (LOCK_SUCCESS.equals(result)) {
     return value;
    }
    try {
     Thread.sleep(10);
    } catch (InterruptedException e) {
     Thread.currentThread().interrupt();
    }
   }
  } catch (Exception ex) {
   log.error("lock error", ex);
  }
  return null;
 }
 public boolean release(String value) {
  if (value == null) {
   return false;
  }
  // 判断key存在并且删除key必须是一个原子操作
  // 且谁拥有锁，谁释放
  String script = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
  Object result = new Object();
  try {
   result = jedis.eval(script, Collections.singletonList(key),
     Collections.singletonList(value));
   if (RELEASE_SUCCESS.equals(result)) {
    log.info("release lock success, value:{}", value);
    return true;
   }
  } catch (Exception e) {
   log.error("release lock error", e);
  } finally {
   if (jedis != null) {
    jedis.close();
   }
  }
  log.info("release lock failed, value:{}, result:{}", value, result);
  return false;
 }
}
```
网上说的通过如下命令获取锁
```java
SETNX resource_name my_random_value
EXPIRE resource_name 30
```
由于这两个命令不是原子的，如果客户端在执行完SETNX后crash了，那么久没有机会执行EXPIRE了，导致它一直持有这个锁，其他的客户端久永远获取不到这个锁了。

