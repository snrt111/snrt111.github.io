---
title: Redis详解
date: 2026-05-19 14:00:00
tags: [Redis, 缓存, 面试题, 分布式, NoSQL]
categories:
  - 面试题
---

# Redis高频面试题30道（含详细答案解析）

Redis作为高性能的键值存储系统，在现代分布式系统中扮演着核心角色。本文整理了Redis领域最高频的30道面试题，涵盖数据结构、持久化、集群、缓存问题等核心知识点，帮助你在面试中脱颖而出。

---

## 一、基础概念篇

### 1. Redis是什么？它有哪些特点？

**答案：**

Redis（Remote Dictionary Server）是一个开源的、基于内存的高性能键值对数据库。

**核心特点：**
- **高性能**：读写速度可达10万+ QPS，纯内存操作
- **丰富的数据类型**：支持String、Hash、List、Set、Sorted Set、Bitmap、HyperLogLog、Geo等
- **持久化支持**：支持RDB快照和AOF日志两种持久化方式
- **高可用**：支持主从复制、Sentinel哨兵、Cluster集群模式
- **原子性操作**：所有操作都是原子性的，支持事务
- **发布订阅**：支持Pub/Sub消息模式
- **Lua脚本**：支持Lua脚本执行复杂操作

**使用场景：**
- 缓存系统、会话存储、排行榜、计数器、分布式锁、消息队列、实时系统

---

### 2. Redis支持哪些数据类型？各自的使用场景是什么？

**答案：**

| 数据类型 | 说明 | 典型使用场景 |
|---------|------|-------------|
| **String** | 字符串，最大512MB | 缓存、计数器、分布式锁、Session |
| **Hash** | 键值对集合 | 存储对象（如用户信息）、购物车 |
| **List** | 双向链表 | 消息队列、时间线、最新消息列表 |
| **Set** | 无序唯一集合 | 标签系统、共同好友、抽奖 |
| **Sorted Set** | 有序集合（带分数） | 排行榜、延迟队列、范围查询 |
| **Bitmap** | 位图 | 用户签到、在线状态统计 |
| **HyperLogLog** | 基数统计 | UV统计、大规模去重计数 |
| **Geo** | 地理位置 | 附近的人、位置服务 |
| **Stream** | 消息流 | 日志收集、消息队列（Kafka替代品） |

**解决问题：**
- 不同业务场景选择合适的数据结构，提升存储效率和查询性能

---

### 3. Redis是单线程还是多线程？为什么单线程还能这么快？

**答案：**

**Redis 6.0之前是单线程模型**，6.0之后引入多线程但仅用于网络IO读写，命令执行仍是单线程。

**单线程快的原因：**
1. **纯内存操作**：所有数据都在内存中，没有磁盘IO
2. **避免上下文切换**：单线程无需线程切换开销
3. **避免锁竞争**：单线程不存在多线程竞争问题
4. **高效的数据结构**：底层使用跳表、压缩列表等优化结构
5. **IO多路复用**：使用epoll/kqueue处理大量连接

**6.0多线程的作用：**
- 网络IO读写使用多线程，提升网络吞吐量
- 命令执行保持单线程，保证原子性

**使用场景：**
- 高并发读写的缓存场景，利用单线程原子性保证数据一致性

---

## 二、持久化篇

### 4. Redis的持久化机制有哪些？各自的优缺点是什么？

**答案：**

Redis提供两种持久化方式：RDB和AOF。

**RDB（Redis Database）**
- **原理**：定时将内存数据快照保存到磁盘
- **触发方式**：手动（SAVE/BGSAVE）、自动（配置规则）
- **优点**：
  - 文件紧凑，适合备份和恢复
  - 恢复速度快
  - 对性能影响小（子进程执行）
- **缺点**：
  - 可能丢失最后一次快照后的数据
  - 大数据量时fork子进程可能耗时

**AOF（Append Only File）**
- **原理**：记录每个写操作命令，重启时重新执行
- **同步策略**：always、everysec（默认）、no
- **优点**：
  - 数据安全性高，最多丢失1秒数据
  - 支持日志重写（AOF Rewrite）压缩文件
- **缺点**：
  - 文件体积大
  - 恢复速度慢
  - 对性能有一定影响

**混合持久化（Redis 4.0+）**
- AOF文件开头是RDB格式，后面是AOF格式
- 结合两者优点：快速恢复 + 数据安全

**使用场景：**
- 数据可丢失：仅RDB
- 数据不可丢失：AOF或混合持久化
- 生产环境推荐：混合持久化 + 定期RDB备份

---

### 5. AOF重写是什么？为什么要重写？

**答案：**

**AOF重写（AOF Rewrite）**是将现有AOF文件中的命令进行合并优化，生成一个新的、更紧凑的AOF文件。

**重写原理：**
- 读取当前内存中的数据状态
- 用最少的命令重建当前数据集
- 例如：多次INCR合并为一次SET

**重写触发条件：**
- 手动触发：`BGREWRITEAOF`
- 自动触发：
  - `auto-aof-rewrite-min-size`：AOF文件最小大小（默认64MB）
  - `auto-aof-rewrite-percentage`：增长率（默认100%）

**重写过程（Copy-on-Write）：**
1. fork子进程读取当前内存数据
2. 子进程写入新AOF文件
3. 父进程继续处理命令，写入AOF缓冲区
4. 子进程完成后，父进程追加缓冲区内容
5. 原子替换旧AOF文件

**解决问题：**
- 减少AOF文件体积，节省磁盘空间
- 加速Redis启动恢复速度

---

## 三、缓存问题篇

### 6. 什么是缓存穿透？如何解决？

**答案：**

**缓存穿透**是指查询一个不存在的数据，缓存中没有，数据库中也没有，导致每次请求都打到数据库。

**产生原因：**
- 恶意攻击：构造大量不存在的key进行查询
- 业务误用：查询已被删除的数据

**解决方案：**

1. **缓存空值（Cache Null）**
   - 数据库查询为空时，也将空值缓存（设置较短过期时间）
   - 优点：简单有效
   - 缺点：缓存大量空值占用空间

2. **布隆过滤器（Bloom Filter）**
   - 在缓存前加一层布隆过滤器，快速判断key是否存在
   - 优点：内存占用小，查询快
   - 缺点：有一定误判率，不能删除元素

3. **参数校验**
   - 对请求参数进行合法性校验，拦截明显非法的请求

4. **限流熔断**
   - 对异常请求进行限流，防止系统被压垮

**使用场景：**
- 电商商品查询、用户信息查询等需要防止恶意攻击的场景

---

### 7. 什么是缓存击穿？如何解决？

**答案：**

**缓存击穿**是指某个热点key在高并发下突然过期，大量请求同时打到数据库。

**产生原因：**
- 热点数据过期时间设置不合理
- 高并发场景下热点key同时失效

**解决方案：**

1. **互斥锁（Mutex Lock）**
   ```java
   String value = redis.get(key);
   if (value == null) {
       // 获取分布式锁
       if (redis.setnx(lockKey, "1", 10)) {
           try {
               value = db.get(key);
               redis.set(key, value, expireTime);
           } finally {
               redis.del(lockKey);
           }
       } else {
           // 其他线程等待后重试
           Thread.sleep(100);
           return get(key); // 重试
       }
   }
   ```

2. **逻辑过期（永不过期）**
   - 不设置TTL，在value中存储逻辑过期时间
   - 发现过期时异步更新缓存
   - 优点：不会阻塞请求
   - 缺点：可能读到旧数据

3. **热点key永不过期**
   - 对热点key不设置过期时间，通过后台任务更新

**使用场景：**
- 秒杀活动中的商品库存、热点新闻内容等高并发读取场景

---

### 8. 什么是缓存雪崩？如何解决？

**答案：**

**缓存雪崩**是指大量缓存key在同一时间过期，或者Redis宕机，导致大量请求直接打到数据库，数据库压力激增甚至宕机。

**产生原因：**
- 缓存服务器重启或宕机
- 大量key设置了相同的过期时间

**解决方案：**

1. **过期时间加随机值**
   ```java
   // 基础过期时间 + 随机偏移
   int expireTime = baseTime + random.nextInt(1000);
   ```

2. **多级缓存**
   - 本地缓存（Caffeine/Guava）+ Redis + 数据库
   - 本地缓存作为兜底

3. **缓存高可用**
   - Redis主从复制、哨兵模式、集群模式
   - 保证缓存服务的高可用性

4. **熔断降级**
   - 数据库压力过大时，启动熔断，返回默认值或错误提示
   - 使用Sentinel、Hystrix等熔断组件

5. **提前预热**
   - 系统启动或低峰期，提前加载热点数据到缓存

**使用场景：**
- 电商大促、活动开始时的缓存系统保护

---

### 9. 如何保证缓存与数据库的一致性？

**答案：**

**常见方案及问题：**

**方案一：先更新数据库，再更新缓存**
- 问题：并发环境下，可能出现缓存脏数据

**方案二：先更新数据库，再删除缓存（Cache Aside）**
- 问题：极端情况下仍有短暂不一致
- 这是目前最常用的方案

**方案三：先删除缓存，再更新数据库**
- 问题：如果更新失败，缓存已被删除，导致缓存击穿

**推荐方案：延迟双删**
```java
// 1. 删除缓存
redis.del(key);
// 2. 更新数据库
db.update(data);
// 3. 延迟一段时间再次删除缓存
Thread.sleep(500);
redis.del(key);
```

**更可靠的方案：消息队列 + 订阅binlog**
- 使用Canal订阅MySQL binlog
- 数据变更后异步更新/删除缓存
- 最终一致性保证

**使用场景：**
- 对一致性要求高的金融、电商库存等场景

---

## 四、分布式锁篇

### 10. 如何用Redis实现分布式锁？

**答案：**

**基础实现（SETNX + EXPIRE）：**
```bash
SET lock_key unique_value NX EX 30
```

**Java实现（Redisson）：**
```java
RLock lock = redisson.getLock("myLock");
try {
    // 尝试加锁，最多等待10秒，锁30秒后自动释放
    boolean locked = lock.tryLock(10, 30, TimeUnit.SECONDS);
    if (locked) {
        // 执行业务逻辑
    }
} finally {
    lock.unlock();
}
```

**分布式锁的关键要素：**

1. **互斥性**：同一时刻只有一个客户端能获取锁
2. **防死锁**：设置过期时间，防止客户端崩溃导致锁无法释放
3. **唯一标识**：value设置唯一标识（UUID），防止误删他人锁
4. **可重入性**：支持同一线程多次获取锁
5. **看门狗机制**：自动续期，防止业务未完成锁过期

**Redisson看门狗原理：**
- 获取锁成功后，启动定时任务，每10秒检查一次
- 如果业务未完成，自动续期到30秒
- 业务完成后，取消定时任务

**使用场景：**
- 库存扣减、订单创建、定时任务调度等需要互斥执行的场景

---

### 11. RedLock是什么？有什么问题？

**答案：**

**RedLock**是Redis作者提出的多节点分布式锁算法，用于解决单点Redis的可靠性问题。

**RedLock算法流程：**
1. 获取当前时间戳
2. 依次向N个独立的Redis节点获取锁（使用相同的key和value）
3. 计算获取锁的总耗时
4. 如果成功获取多数节点（N/2+1）的锁，且总耗时小于锁过期时间，则获取锁成功
5. 如果获取失败，向所有节点发送释放锁命令

**RedLock的问题：**

1. **时钟漂移问题**
   - 依赖系统时钟，如果节点时钟不同步，可能导致锁失效

2. **网络延迟问题**
   - 获取锁的过程中，如果某个节点网络延迟，可能导致判断错误

3. **持久化问题**
   - 如果Redis节点开启持久化，重启后可能恢复已释放的锁

4. **争议性**
   - Martin Kleppmann（分布式系统专家）发文质疑RedLock的安全性
   - 建议使用ZooKeeper或etcd等强一致性协调服务

**实际建议：**
- 大多数场景下单Redis节点+Redisson已足够
- 对可靠性要求极高的场景，考虑使用ZooKeeper/etcd

---

## 五、集群与高可用篇

### 12. Redis主从复制的工作原理是什么？

**答案：**

**主从复制（Replication）**是Redis实现高可用的基础，一个主节点（Master）可以有多个从节点（Slave）。

**复制过程：**

1. **全量同步（Full Resynchronization）**
   - 从节点发送`SYNC`或`PSYNC`命令
   - 主节点执行`BGSAVE`生成RDB文件
   - 主节点将RDB文件发送给从节点
   - 从节点加载RDB文件
   - 主节点将复制期间的写命令发送给从节点

2. **增量同步（Partial Resynchronization）**
   - 使用`PSYNC`命令，基于复制偏移量（offset）和复制积压缓冲区（replication backlog）
   - 如果从节点断开时间不长，只需同步断连期间的命令

**复制方式：**
- **同步复制**：主节点等待从节点确认（影响性能，不常用）
- **异步复制**：主节点不等待从节点（默认，性能好但可能丢数据）

**使用场景：**
- 读写分离：主写从读，扩展读性能
- 数据备份：从节点作为数据备份
- 高可用基础：为哨兵和集群提供基础

---

### 13. Redis哨兵（Sentinel）模式是什么？

**答案：**

**Redis Sentinel**是Redis的高可用解决方案，用于监控主从节点，实现自动故障转移。

**Sentinel的核心功能：**

1. **监控（Monitoring）**
   - 持续检查主从节点是否正常工作

2. **通知（Notification）**
   - 节点故障时，通过API通知管理员或其他应用程序

3. **自动故障转移（Automatic Failover）**
   - 主节点故障时，自动将一个从节点提升为主节点
   - 通知其他从节点修改复制目标
   - 通知客户端更新主节点地址

4. **配置提供（Configuration Provider）**
   - 客户端向Sentinel获取当前主节点地址

**故障转移流程：**
1. 多个Sentinel发现主节点主观下线（SDOWN）
2. 达成客观下线（ODOWN）共识
3. 选举Leader Sentinel
4. Leader选择一个从节点提升为主节点
5. 通知其他从节点复制新主节点
6. 原主节点恢复后变为从节点

**使用场景：**
- 中小规模Redis部署的高可用方案
- 需要自动故障转移但数据量不大的场景

---

### 14. Redis Cluster集群的工作原理是什么？

**答案：**

**Redis Cluster**是Redis的分布式解决方案，提供数据分片和高可用功能。

**核心特性：**

1. **数据分片（Sharding）**
   - 使用哈希槽（Hash Slot）将数据分布到多个节点
   - 共16384个哈希槽（0-16383）
   - 每个key通过`CRC16(key) % 16384`计算所属槽

2. **节点通信**
   - 使用Gossip协议进行节点间通信
   - 每个节点都知道集群中其他节点的槽分配信息

3. **高可用**
   - 每个主节点可以有多个从节点
   - 主节点故障时，从节点自动提升为主节点

**集群架构示例：**
```
Master A (0-5460)  <-- Slave A1, Slave A2
Master B (5461-10922) <-- Slave B1, Slave B2  
Master C (10923-16383) <-- Slave C1, Slave C2
```

**MOVED和ASK重定向：**
- **MOVED**：key永久不在当前节点，客户端需要更新槽映射
- **ASK**：key正在迁移中，临时访问目标节点

**使用场景：**
- 大规模数据存储（超过单节点内存容量）
- 高并发读写需要水平扩展的场景
- 需要自动故障转移的大规模部署

---

### 15. Redis Cluster和Redis Sentinel有什么区别？

**答案：**

| 特性 | Redis Sentinel | Redis Cluster |
|------|----------------|---------------|
| **数据分片** | 不支持，所有数据在主从节点间全量复制 | 支持，数据分布在多个主节点 |
| **存储容量** | 受限于单节点内存 | 可水平扩展，支持TB级数据 |
| **写入性能** | 单点写入，无法扩展 | 多点写入，可水平扩展 |
| **复杂度** | 相对较低 | 相对较高 |
| **客户端要求** | 普通客户端即可 | 需要支持Cluster协议的客户端 |
| **适用规模** | 中小型应用 | 大型应用 |

**选择建议：**
- 数据量小（<10GB）、读多写少：Sentinel
- 数据量大、需要水平扩展：Cluster

---

## 六、性能优化篇

### 16. Redis内存满了怎么办？有哪些内存淘汰策略？

**答案：**

**当Redis内存达到上限（maxmemory）时，会触发内存淘汰策略。**

**内存淘汰策略（maxmemory-policy）：**

1. **noeviction**（默认）
   - 不淘汰数据，新写入操作返回错误
   - 适用于不允许数据丢失的场景

2. **allkeys-lru**
   - 在所有key中使用LRU算法淘汰最近最少使用的
   - 最常用的策略

3. **volatile-lru**
   - 只在设置了过期时间的key中使用LRU淘汰

4. **allkeys-lfu**（Redis 4.0+）
   - 在所有key中使用LFU算法淘汰使用频率最低的

5. **volatile-lfu**（Redis 4.0+）
   - 只在设置了过期时间的key中使用LFU淘汰

6. **allkeys-random**
   - 随机淘汰所有key

7. **volatile-random**
   - 随机淘汰设置了过期时间的key

8. **volatile-ttl**
   - 淘汰即将过期的key（TTL最小的）

**LRU vs LFU：**
- **LRU（Least Recently Used）**：关注最后一次访问时间
- **LFU（Least Frequently Used）**：关注访问频率，更适合热点数据场景

**使用场景：**
- 缓存场景推荐：allkeys-lru或allkeys-lfu
- 需要保留重要数据：volatile-lru，重要key不设置过期时间

---

### 17. 如何排查Redis性能问题？

**答案：**

**1. 慢查询分析**
```bash
# 设置慢查询阈值（微秒）
CONFIG SET slowlog-log-slower-than 10000
# 查看慢查询日志
SLOWLOG GET 10
```

**2. 监控关键指标**
- `redis-cli INFO`：查看内存、连接、命令统计等
- `redis-cli --latency`：检测延迟
- `redis-cli MONITOR`：实时监控命令（生产环境慎用）

**3. 常见性能问题及解决：**

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 高CPU | 复杂命令（KEYS、HGETALL） | 禁用KEYS，使用SCAN；避免大数据量操作 |
| 高内存 | 内存碎片、大key | 重启或使用jemalloc；拆分大key |
| 高延迟 | 持久化fork、AOF同步 | 优化持久化配置；使用SSD |
| 连接数过高 | 连接未释放 | 使用连接池；检查连接泄露 |

**4. 大key排查**
```bash
# 扫描大key
redis-cli --bigkeys
# 分析内存使用
redis-cli MEMORY USAGE key_name
```

**使用场景：**
- 生产环境性能调优、故障排查

---

### 18. 什么是Pipeline？有什么用？

**答案：**

**Pipeline**是Redis提供的一种批量执行命令的机制，允许客户端一次性发送多个命令，然后一次性接收结果。

**工作原理：**
- 普通模式：每个命令需要一次RTT（Round Trip Time）
- Pipeline模式：多个命令打包发送，只需一次RTT

**性能对比：**
- 10000个命令，普通模式：10000次RTT
- 10000个命令，Pipeline模式：1次RTT

**Java示例：**
```java
redisTemplate.executePipelined(new RedisCallback<Object>() {
    @Override
    public Object doInRedis(RedisConnection connection) {
        for (int i = 0; i < 10000; i++) {
            connection.set(("key" + i).getBytes(), ("value" + i).getBytes());
        }
        return null;
    }
});
```

**注意事项：**
- Pipeline不是原子操作，中间可能插入其他命令
- 命令过多时会占用大量内存，建议分批发送（如每1000个一批）
- 不支持事务（如需事务使用MULTI/EXEC）

**使用场景：**
- 批量写入数据、批量查询、数据迁移

---

### 19. 什么是Redis事务？和Lua脚本有什么区别？

**答案：**

**Redis事务（MULTI/EXEC）：**
```bash
MULTI
SET key1 value1
SET key2 value2
EXEC
```

**事务特性：**
- **批量执行**：命令按顺序执行，不会被其他命令插入
- **不保证原子性**：如果中间命令出错，后续命令仍会继续执行
- **没有回滚**：不支持事务回滚

**Lua脚本：**
```bash
EVAL "redis.call('SET', KEYS[1], ARGV[1]); redis.call('INCR', KEYS[2])" 2 key1 key2 value1
```

**Lua脚本特性：**
- **原子性**：整个脚本作为一个命令执行，不会被中断
- **功能强大**：支持逻辑判断、循环等复杂操作
- **减少网络往返**：多个操作在一个脚本中完成

**对比：**

| 特性 | 事务 | Lua脚本 |
|------|------|---------|
| 原子性 | 部分保证 | 完全保证 |
| 回滚 | 不支持 | 不支持 |
| 复杂逻辑 | 不支持 | 支持 |
| 性能 | 一般 | 更好 |

**使用场景：**
- 简单批量操作：事务
- 复杂逻辑、需要原子性：Lua脚本

---

## 七、应用场景篇

### 20. Redis如何实现排行榜功能？

**答案：**

**使用Sorted Set（ZSet）实现排行榜：**

```bash
# 添加用户分数
ZADD leaderboard 100 "user1"
ZADD leaderboard 200 "user2"
ZADD leaderboard 150 "user3"

# 获取前10名（分数从高到低）
ZREVRANGE leaderboard 0 9 WITHSCORES

# 获取用户排名
ZREVRANK leaderboard "user1"

# 获取用户分数
ZSCORE leaderboard "user1"

# 增加用户分数
ZINCRBY leaderboard 10 "user1"
```

**带时间戳的排行榜（同分按时间排序）：**
```bash
# 分数 = 实际分数 * 10000000000 + (9999999999 - 时间戳)
# 这样同分情况下，先达到的排名靠前
ZADD leaderboard 1009999999999 "user1"
```

**多维度排行榜：**
- 使用多个ZSet，如`leaderboard:daily`、`leaderboard:weekly`
- 使用Hash存储用户详细信息

**使用场景：**
- 游戏排行榜、积分排名、热销榜单

---

### 21. Redis如何实现限流？

**答案：**

**方案一：计数器法（固定窗口）**
```bash
# 每分钟限制100次
INCR user:123:api_limit
EXPIRE user:123:api_limit 60
```
- 缺点：窗口切换时可能出现2倍流量突刺

**方案二：滑动窗口（ZSet实现）**
```java
// 使用ZSet记录每次请求的时间戳
ZADD rate_limit:user:123 current_timestamp current_timestamp
// 移除窗口外的记录
ZREMRANGEBYSCORE rate_limit:user:123 0 (current_timestamp - 60)
// 统计当前窗口内的请求数
ZCARD rate_limit:user:123
// 设置过期时间
EXPIRE rate_limit:user:123 60
```

**方案三：令牌桶（Redis + Lua）**
```lua
local key = KEYS[1]
local rate = tonumber(ARGV[1])  -- 每秒产生令牌数
local capacity = tonumber(ARGV[2])  -- 桶容量
local now = tonumber(ARGV[3])  -- 当前时间戳

local tokens = redis.call('HMGET', key, 'tokens', 'last_time')
local last_tokens = tonumber(tokens[1]) or capacity
local last_time = tonumber(tokens[2]) or now

-- 计算新令牌数
local delta = math.max(0, now - last_time)
local new_tokens = math.min(capacity, last_tokens + delta * rate)

if new_tokens >= 1 then
    new_tokens = new_tokens - 1
    redis.call('HMSET', key, 'tokens', new_tokens, 'last_time', now)
    return 1  -- 允许通过
else
    redis.call('HSET', key, 'last_time', now)
    return 0  -- 拒绝
end
```

**使用场景：**
- API限流、用户操作频率限制、防止暴力破解

---

### 22. Redis如何实现消息队列？

**答案：**

**方案一：List实现（简单队列）**
```bash
# 生产者
LPUSH queue:message "task_data"

# 消费者（阻塞弹出）
BRPOP queue:message 30
```
- 优点：简单可靠
- 缺点：不支持多播、无ACK机制

**方案二：Pub/Sub（发布订阅）**
```bash
# 订阅者
SUBSCRIBE channel1

# 发布者
PUBLISH channel1 "message"
```
- 优点：支持多播
- 缺点：消息不持久化，消费者离线期间的消息会丢失

**方案三：Stream（推荐，Redis 5.0+）**
```bash
# 生产者
XADD mystream * field1 value1 field2 value2

# 消费者（消费者组）
XGROUP CREATE mystream mygroup $
XREADGROUP GROUP mygroup consumer1 COUNT 1 STREAMS mystream >

# 确认消费
XACK mystream mygroup message_id
```
- 优点：支持持久化、消费者组、ACK机制、消息回溯
- 适用：需要可靠消息传递的场景

**使用场景：**
- 简单任务队列：List
- 实时通知：Pub/Sub
- 可靠消息系统：Stream

---

### 23. Redis如何实现分布式会话（Session）？

**答案：**

**Spring Session + Redis实现：**

```java
// 依赖
// spring-session-data-redis

// 配置
@Configuration
@EnableRedisHttpSession(maxInactiveIntervalInSeconds = 1800)
public class SessionConfig {
    @Bean
    public LettuceConnectionFactory connectionFactory() {
        return new LettuceConnectionFactory();
    }
}
```

**原理：**
1. 用户登录后，Session数据存储到Redis
2. Session ID通过Cookie返回给客户端
3. 后续请求携带Session ID，从Redis获取Session数据
4. 多台应用服务器共享同一份Session数据

**Session数据结构：**
```bash
# Spring Session在Redis中的存储
HSET spring:session:sessions:session_id sessionAttr:username "zhangsan"
HSET spring:session:sessions:session_id creationTime "1234567890"
HSET spring:session:sessions:session_id maxInactiveInterval "1800"
```

**使用场景：**
- 集群环境下的用户登录状态保持
- 多服务共享用户会话信息

---

### 24. Redis如何实现附近的人功能？

**答案：**

**使用Redis Geo（地理位置）功能：**

```bash
# 添加位置
GEOADD locations 116.3974 39.9092 "user1"  # 经度 纬度 成员
GEOADD locations 116.4074 39.9192 "user2"
GEOADD locations 116.3874 39.8992 "user3"

# 查询附近的人（半径5公里）
GEORADIUS locations 116.3974 39.9092 5 km WITHDIST WITHCOORD COUNT 10

# 查询指定成员附近的人
GEORADIUSBYMEMBER locations "user1" 5 km WITHDIST

# 计算两个成员之间的距离
GEODIST locations "user1" "user2" km
```

**Java实现：**
```java
// 添加位置
redisTemplate.opsForGeo().add("locations", 
    new Point(116.3974, 39.9092), "user1");

// 查询附近
GeoResults<RedisGeoCommands.GeoLocation<String>> results = 
    redisTemplate.opsForGeo().radius("locations",
        new Circle(new Point(116.3974, 39.9092), 
                   new Distance(5, Metrics.KILOMETERS)),
        RedisGeoCommands.GeoRadiusCommandArgs.newGeoRadiusArgs()
            .includeDistance().includeCoordinates().limit(10));
```

**底层实现：**
- 使用GeoHash算法将二维坐标编码为字符串
- 使用Sorted Set存储，GeoHash作为score

**使用场景：**
- 附近的人、附近商家、地理位置服务

---

## 八、高级特性篇

### 25. Redis的Bitmap有什么使用场景？

**答案：**

**Bitmap**是String类型的扩展，可以对每个bit进行独立操作。

**基本操作：**
```bash
# 设置第100位为1
SETBIT user:123:sign 100 1

# 获取第100位的值
GETBIT user:123:sign 100

# 统计1的个数（签到天数）
BITCOUNT user:123:sign

# 查找第一个1的位置
BITPOS user:123:sign 1

# 位运算（AND、OR、XOR、NOT）
BITOP AND result user:1:sign user:2:sign
```

**使用场景：**

1. **用户签到**
   - 每个用户一年只需要365 bits ≈ 46 bytes
   - 1000万用户一年仅需约440MB

2. **在线状态统计**
   ```bash
   SETBIT online_users 10001 1  # 用户10001上线
   SETBIT online_users 10001 0  # 用户10001下线
   BITCOUNT online_users        # 统计在线人数
   ```

3. **布隆过滤器**
   - 使用多个Bitmap实现简单的布隆过滤器

4. **用户标签系统**
   ```bash
   # 每个bit代表一个标签
   SETBIT user:123:tags 0 1  # 标签0：VIP
   SETBIT user:123:tags 1 0  # 标签1：新用户
   ```

**优势：**
- 极低的存储成本
- 高效的位运算

---

### 26. Redis的HyperLogLog是什么？

**答案：**

**HyperLogLog**是一种概率数据结构，用于高效地统计基数（不重复元素的数量），误差率约0.81%。

**基本操作：**
```bash
# 添加元素
PFADD visitors user1 user2 user3 user1

# 统计基数（返回3）
PFCOUNT visitors

# 合并多个HyperLogLog
PFMERGE total_visitors visitors_day1 visitors_day2
```

**特点：**
- 固定内存占用：每个HyperLogLog仅需12KB
- 可统计2^64个不同元素
- 不存储实际元素，只存储基数信息

**对比：**

| 方案 | 内存占用（1000万UV） | 精度 |
|------|---------------------|------|
| Set | 约600MB | 精确 |
| Bitmap | 约1.2MB | 精确 |
| HyperLogLog | 12KB | 近似（0.81%误差） |

**使用场景：**
- 网站UV统计、搜索关键词去重计数、广告点击去重
- 对精度要求不高，但数据量巨大的场景

---

### 27. Redis的Scan命令是什么？与KEYS有什么区别？

**答案：**

**KEYS的问题：**
```bash
KEYS user:*  # 阻塞操作，大数据量时会导致Redis卡顿
```

**SCAN命令：**
```bash
# 迭代器方式遍历key
SCAN 0 MATCH user:* COUNT 100
SCAN 17 MATCH user:* COUNT 100  # 使用返回的cursor继续
```

**SCAN特点：**
- **非阻塞**：基于游标的迭代器，不会阻塞服务器
- **增量式**：每次只返回部分结果
- **状态保存在客户端**：游标由客户端保存

**注意事项：**
- 迭代过程中，如果数据有变化，可能返回重复或遗漏
- COUNT是提示值，实际返回数量可能不同
- 需要一直迭代直到游标返回0

**其他迭代命令：**
- `HSCAN`：遍历Hash
- `SSCAN`：遍历Set
- `ZSCAN`：遍历Sorted Set

**使用场景：**
- 生产环境遍历key、数据迁移、批量处理

---

### 28. Redis的内存碎片化问题如何解决？

**答案：**

**内存碎片产生原因：**
- 频繁增删key导致内存分配不连续
- Redis使用jemalloc分配器，按固定大小分配内存

**查看内存碎片：**
```bash
INFO memory
# mem_fragmentation_ratio = used_memory_rss / used_memory
# > 1.5 表示碎片较严重
```

**解决方案：**

1. **自动内存碎片整理（Redis 4.0+）**
```bash
# 开启自动碎片整理
CONFIG SET activedefrag yes

# 配置参数
CONFIG SET active-defrag-threshold-lower 10   # 碎片率超过10%开始整理
CONFIG SET active-defrag-threshold-upper 100  # 碎片率超过100%全力整理
CONFIG SET active-defrag-cycle-min 5          # 最小CPU占用
CONFIG SET active-defrag-cycle-max 75         # 最大CPU占用
```

2. **手动重启**
   - 数据持久化后重启Redis，内存重新分配

3. **数据重写**
   - 使用`DEBUG RELOAD`或主从切换

**使用场景：**
- 长期运行的Redis实例、频繁增删数据的场景

---

### 29. Redis的大key问题如何排查和解决？

**答案：**

**大key的定义：**
- String类型：value > 10KB
- 集合类型：元素数量 > 5000

**排查方法：**

1. **redis-cli --bigkeys**
```bash
redis-cli --bigkeys
# 扫描每种数据类型的最大key
```

2. **MEMORY USAGE命令**
```bash
MEMORY USAGE key_name
```

3. **rdb-tools分析RDB文件**
```bash
rdb -c memory dump.rdb -l largest 100 > large_keys.csv
```

**大key的危害：**
- 阻塞Redis（删除大key时）
- 网络拥塞（读取大key时）
- 主从同步延迟

**解决方案：**

1. **拆分大key**
   ```bash
   # 将大Hash拆分成多个小Hash
   # user:1000 -> user:1000:profile, user:1000:orders
   ```

2. **异步删除（Redis 4.0+）**
   ```bash
   UNLINK big_key  # 异步删除，非阻塞
   ```

3. **渐进式删除**
   ```bash
   # 对于大Hash，使用HSCAN分批删除
   HSCAN big_hash 0 COUNT 100
   HDEL big_hash field1 field2 ...
   ```

**使用场景：**
- 生产环境性能优化、防止Redis阻塞

---

### 30. Redis 6.0/7.0有哪些重要新特性？

**答案：**

**Redis 6.0新特性：**

1. **多线程IO**
   - 网络读写使用多线程，提升吞吐量
   - 命令执行仍为单线程

2. **ACL（访问控制列表）**
   ```bash
   ACL SETUSER alice on >password ~cached:* +get +set
   ```

3. **SSL/TLS支持**
   - 原生支持加密连接

4. **RESP3协议**
   - 新的Redis序列化协议

**Redis 7.0新特性：**

1. **Function（函数）**
   - 持久化的Lua函数，重启后仍然存在
   ```bash
   FUNCTION LOAD "#!lua name=mylib
   redis.register_function('myfunc', function(keys, args)
       return redis.call('GET', keys[1])
   end)"
   ```

2. **Sharded Pub/Sub**
   - 集群模式下Pub/Sub的优化

3. **多部分AOF**
   - AOF文件分为基础文件和增量文件，便于管理

4. **性能优化**
   - 内存效率提升
   - 命令执行优化

**使用场景：**
- 新特性可以提升安全性、可维护性和性能，建议关注版本升级

---

## 总结

Redis作为高性能缓存和存储系统，掌握其核心原理和最佳实践对于Java开发者至关重要。本文涵盖了Redis的数据结构、持久化、集群、缓存问题、分布式锁、性能优化等核心知识点，建议结合实际项目进行深入理解和实践。

**学习建议：**
1. 动手搭建Redis主从、哨兵、集群环境
2. 使用Redisson实现分布式锁和限流
3. 结合Spring Boot和Spring Data Redis进行实战
4. 关注Redis官方文档，了解最新特性
