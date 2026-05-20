---
title: Redis详解
date: 2026-05-19 14:00:00
tags: [Redis, 缓存, 面试题, 分布式, NoSQL]
categories:
  - 面试题
---

# Redis详解

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

**为什么使用Sorted Set？**
- Sorted Set底层使用**跳表（Skip List）** + **哈希表**实现
- 跳表保证范围查询效率为O(log N)，哈希表保证单点查询为O(1)
- 天然支持按分数排序，非常适合排行榜场景

**基础实现：**

```bash
# 添加用户分数
ZADD leaderboard 100 "user1"
ZADD leaderboard 200 "user2"
ZADD leaderboard 150 "user3"

# 获取前10名（分数从高到低）
ZREVRANGE leaderboard 0 9 WITHSCORES

# 获取用户排名（从0开始）
ZREVRANK leaderboard "user1"

# 获取用户分数
ZSCORE leaderboard "user1"

# 增加用户分数（原子操作，线程安全）
ZINCRBY leaderboard 10 "user1"

# 获取指定分数范围的成员
ZREVRANGEBYSCORE leaderboard 200 100 WITHSCORES

# 移除成员
ZREM leaderboard "user1"
```

**同分情况下的时间排序方案：**

```bash
# 方案：将时间戳编码到分数的小数部分
# 分数 = 实际分数 + (1 - 时间戳/10^10) * 0.0000000001
# 这样同分情况下，先达到的时间戳大，小数部分小，排名靠前

# 实际应用示例（假设当前时间戳为1700000000）
ZADD leaderboard 100.8299999999 "user1"  # 先达到100分
ZADD leaderboard 100.8199999999 "user2"  # 后达到100分
```

**Java完整实现：**

```java
@Service
public class LeaderboardService {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    private static final String LEADERBOARD_KEY = "leaderboard:game";
    private static final long TIME_FACTOR = 1_000_000_000L;
    
    /**
     * 添加分数（支持同分按时间排序）
     */
    public void addScore(String userId, double score) {
        // 编码时间戳到小数部分
        long timestamp = System.currentTimeMillis();
        double encodedScore = encodeScore(score, timestamp);
        redisTemplate.opsForZSet().add(LEADERBOARD_KEY, userId, encodedScore);
    }
    
    /**
     * 编码分数：整数部分为实际分数，小数部分为时间编码
     */
    private double encodeScore(double score, long timestamp) {
        // 时间戳越大（越新），小数部分越小，排名越靠后
        double timePart = (TIME_FACTOR - timestamp % TIME_FACTOR) / 1e10;
        return score + timePart;
    }
    
    /**
     * 获取前N名
     */
    public List<LeaderboardItem> getTopN(int n) {
        Set<ZSetOperations.TypedTuple<String>> tuples = 
            redisTemplate.opsForZSet()
                .reverseRangeWithScores(LEADERBOARD_KEY, 0, n - 1);
        
        List<LeaderboardItem> result = new ArrayList<>();
        int rank = 1;
        for (ZSetOperations.TypedTuple<String> tuple : tuples) {
            result.add(new LeaderboardItem(
                rank++,
                tuple.getValue(),
                Math.floor(tuple.getScore())  // 取整数部分
            ));
        }
        return result;
    }
    
    /**
     * 获取用户排名和分数
     */
    public UserRank getUserRank(String userId) {
        Long rank = redisTemplate.opsForZSet()
            .reverseRank(LEADERBOARD_KEY, userId);
        Double score = redisTemplate.opsForZSet()
            .score(LEADERBOARD_KEY, userId);
        
        if (rank == null || score == null) {
            return null;
        }
        
        return new UserRank(
            rank + 1,  // 排名从1开始
            Math.floor(score)  // 取整数部分
        );
    }
    
    /**
     * 获取用户周边排名（如前后各5名）
     */
    public List<LeaderboardItem> getNearbyRank(String userId, int range) {
        Long rank = redisTemplate.opsForZSet()
            .reverseRank(LEADERBOARD_KEY, userId);
        if (rank == null) {
            return Collections.emptyList();
        }
        
        long start = Math.max(0, rank - range);
        long end = rank + range;
        
        return getRange(start, end);
    }
    
    /**
     * 批量增加分数（使用Pipeline提升性能）
     */
    public void batchAddScores(Map<String, Double> userScores) {
        redisTemplate.executePipelined((RedisCallback<Object>) connection -> {
            for (Map.Entry<String, Double> entry : userScores.entrySet()) {
                double encodedScore = encodeScore(entry.getValue(), 
                    System.currentTimeMillis());
                connection.zSetCommands().zAdd(
                    LEADERBOARD_KEY.getBytes(),
                    encodedScore,
                    entry.getKey().getBytes()
                );
            }
            return null;
        });
    }
}

// 数据类
@Data
public class LeaderboardItem {
    private int rank;
    private String userId;
    private double score;
}

@Data
@AllArgsConstructor
public class UserRank {
    private long rank;
    private double score;
}
```

**多维度排行榜设计：**

```java
@Service
public class MultiDimensionLeaderboardService {
    
    private static final String DAILY_KEY = "leaderboard:daily:%s";
    private static final String WEEKLY_KEY = "leaderboard:weekly:%s";
    private static final String MONTHLY_KEY = "leaderboard:monthly:%s";
    private static final String ALL_TIME_KEY = "leaderboard:all_time";
    
    /**
     * 更新所有维度的排行榜
     */
    public void updateAllLeaderboards(String userId, double score) {
        String today = LocalDate.now().toString();
        String week = getWeekKey();
        String month = getMonthKey();
        
        // 使用Pipeline批量更新
        redisTemplate.executePipelined((RedisCallback<Object>) connection -> {
            long timestamp = System.currentTimeMillis();
            double encodedScore = encodeScore(score, timestamp);
            
            // 日榜
            connection.zSetCommands().zIncrBy(
                String.format(DAILY_KEY, today).getBytes(),
                encodedScore,
                userId.getBytes()
            );
            
            // 周榜
            connection.zSetCommands().zIncrBy(
                String.format(WEEKLY_KEY, week).getBytes(),
                encodedScore,
                userId.getBytes()
            );
            
            // 月榜
            connection.zSetCommands().zIncrBy(
                String.format(MONTHLY_KEY, month).getBytes(),
                encodedScore,
                userId.getBytes()
            );
            
            // 总榜
            connection.zSetCommands().zIncrBy(
                ALL_TIME_KEY.getBytes(),
                encodedScore,
                userId.getBytes()
            );
            
            return null;
        });
    }
    
    /**
     * 设置排行榜过期时间（自动清理）
     */
    @Scheduled(cron = "0 0 0 * * ?")  // 每天凌晨执行
    public void setExpiration() {
        String yesterday = LocalDate.now().minusDays(1).toString();
        String key = String.format(DAILY_KEY, yesterday);
        redisTemplate.expire(key, 7, TimeUnit.DAYS);  // 保留7天
    }
}
```

**性能优化建议：**

| 优化点 | 方案 | 效果 |
|--------|------|------|
| 大数据量 | 分片存储（按用户ID哈希） | 降低单个ZSet大小 |
| 高并发写入 | 使用Pipeline批量操作 | 减少网络RTT |
| 冷热分离 | 活跃榜单放Redis，历史数据归档 | 节省内存 |
| 缓存排名 | 定期计算并缓存用户排名 | 减少实时计算压力 |

**使用场景：**
- **游戏排行榜**：实时更新玩家积分排名
- **电商热销榜**：按销量/销售额排序的商品榜单
- **金融收益率排行**：按收益率排序的理财产品
- **社交影响力榜**：按粉丝数/互动数排序的用户榜单

---

### 21. Redis如何实现计数器和限流？

**答案：**

## 一、计数器实现

**基础计数器（String类型）：**

```bash
# 增加计数
INCR view_count:article:1001

# 增加指定数量
INCRBY view_count:article:1001 5

# 减少计数
DECR view_count:article:1001

# 设置过期时间（24小时后过期）
EXPIRE view_count:article:1001 86400

# 获取当前值
GET view_count:article:1001
```

**Hash类型计数器（多维度统计）：**

```bash
# 统计文章的多维度数据
HINCRBY article:1001:stats view_count 1
HINCRBY article:1001:stats like_count 1
HINCRBY article:1001:stats comment_count 1

# 获取所有统计
HGETALL article:1001:stats
```

**Java计数器服务实现：**

```java
@Service
public class CounterService {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    /**
     * 增加计数
     */
    public Long increment(String counterKey) {
        return redisTemplate.opsForValue().increment(counterKey);
    }
    
    /**
     * 增加计数并设置过期时间
     */
    public Long incrementWithExpire(String counterKey, long delta, 
                                     long timeout, TimeUnit unit) {
        Long value = redisTemplate.opsForValue().increment(counterKey, delta);
        redisTemplate.expire(counterKey, timeout, unit);
        return value;
    }
    
    /**
     * 获取当前计数
     */
    public Long getCount(String counterKey) {
        String value = redisTemplate.opsForValue().get(counterKey);
        return value != null ? Long.parseLong(value) : 0L;
    }
    
    /**
     * 批量增加计数（Pipeline优化）
     */
    public void batchIncrement(Map<String, Long> counterMap) {
        redisTemplate.executePipelined((RedisCallback<Object>) connection -> {
            for (Map.Entry<String, Long> entry : counterMap.entrySet()) {
                connection.stringCommands().incrBy(
                    entry.getKey().getBytes(),
                    entry.getValue()
                );
            }
            return null;
        });
    }
}
```

## 二、限流算法详解

| 算法 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| **固定窗口** | 实现简单 | 窗口边界突刺 | 简单限流 |
| **滑动窗口** | 平滑限流 | 内存占用大 | 精确限流 |
| **令牌桶** | 允许突发流量 | 实现复杂 | API网关 |
| **漏桶** | 严格匀速 | 无法应对突发 | 流量整形 |

### 方案一：固定窗口计数器

```java
@Service
public class FixedWindowRateLimiter {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    /**
     * 固定窗口限流
     * @param key 限流标识
     * @param limit 限制次数
     * @param windowSeconds 窗口大小（秒）
     * @return 是否允许通过
     */
    public boolean isAllowed(String key, int limit, int windowSeconds) {
        String redisKey = "rate_limit:fixed:" + key;
        
        // 使用Lua脚本保证原子性
        String luaScript = 
            "local current = redis.call('GET', KEYS[1]);" +
            "if current == false then " +
            "    redis.call('SET', KEYS[1], 1, 'EX', ARGV[2]);" +
            "    return 1;" +
            "end;" +
            "local num = tonumber(current);" +
            "if num < tonumber(ARGV[1]) then " +
            "    redis.call('INCR', KEYS[1]);" +
            "    return 1;" +
            "else " +
            "    return 0;" +
            "end;";
        
        Long result = redisTemplate.execute(
            new DefaultRedisScript<>(luaScript, Long.class),
            Collections.singletonList(redisKey),
            String.valueOf(limit),
            String.valueOf(windowSeconds)
        );
        
        return result != null && result == 1;
    }
}
```

**固定窗口问题示例：**
```
时间线：|-------窗口1-------|-------窗口2-------|
请求：      90次              90次
         在窗口1末尾        在窗口2开头
         
实际：在窗口切换的短时间内有180次请求通过！
```

### 方案二：滑动窗口限流（ZSet实现）

```java
@Service
public class SlidingWindowRateLimiter {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    /**
     * 滑动窗口限流
     * @param key 限流标识
     * @param limit 限制次数
     * @param windowSeconds 窗口大小（秒）
     * @return 是否允许通过
     */
    public boolean isAllowed(String key, int limit, int windowSeconds) {
        String redisKey = "rate_limit:sliding:" + key;
        long now = System.currentTimeMillis();
        long windowStart = now - windowSeconds * 1000;
        
        // 使用Lua脚本保证原子性
        String luaScript = 
            "-- 移除窗口外的旧记录\n" +
            "redis.call('ZREMRANGEBYSCORE', KEYS[1], 0, ARGV[2]);\n" +
            "-- 统计当前窗口内的记录数\n" +
            "local current = redis.call('ZCARD', KEYS[1]);\n" +
            "-- 判断是否超过限制\n" +
            "if tonumber(current) < tonumber(ARGV[1]) then\n" +
            "    redis.call('ZADD', KEYS[1], ARGV[3], ARGV[3]);\n" +
            "    redis.call('EXPIRE', KEYS[1], ARGV[4]);\n" +
            "    return 1;\n" +
            "else\n" +
            "    return 0;\n" +
            "end;";
        
        Long result = redisTemplate.execute(
            new DefaultRedisScript<>(luaScript, Long.class),
            Collections.singletonList(redisKey),
            String.valueOf(limit),           // ARGV[1]: 限制次数
            String.valueOf(windowStart),     // ARGV[2]: 窗口开始时间
            String.valueOf(now),             // ARGV[3]: 当前时间戳
            String.valueOf(windowSeconds)    // ARGV[4]: 过期时间
        );
        
        return result != null && result == 1;
    }
    
    /**
     * 获取当前窗口内的请求次数
     */
    public Long getCurrentCount(String key, int windowSeconds) {
        String redisKey = "rate_limit:sliding:" + key;
        long now = System.currentTimeMillis();
        long windowStart = now - windowSeconds * 1000;
        
        // 清理旧数据
        redisTemplate.opsForZSet()
            .removeRangeByScore(redisKey, 0, windowStart);
        
        // 统计当前数量
        return redisTemplate.opsForZSet().zCard(redisKey);
    }
}
```

### 方案三：令牌桶限流（推荐）

```java
@Service
public class TokenBucketRateLimiter {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    /**
     * 令牌桶限流
     * @param key 限流标识
     * @param capacity 桶容量
     * @param rate 每秒产生令牌数
     * @return 是否允许通过
     */
    public boolean isAllowed(String key, int capacity, double rate) {
        String redisKey = "rate_limit:token:" + key;
        long now = System.currentTimeMillis();
        
        String luaScript = 
            "local key = KEYS[1];\n" +
            "local capacity = tonumber(ARGV[1]);\n" +
            "local rate = tonumber(ARGV[2]);\n" +
            "local now = tonumber(ARGV[3]);\n" +
            "\n" +
            "-- 获取当前令牌数和时间\n" +
            "local tokens = redis.call('HMGET', key, 'tokens', 'last_time');\n" +
            "local last_tokens = tonumber(tokens[1]) or capacity;\n" +
            "local last_time = tonumber(tokens[2]) or now;\n" +
            "\n" +
            "-- 计算新令牌数\n" +
            "local delta = math.max(0, now - last_time) / 1000;\n" +
            "local new_tokens = math.min(capacity, last_tokens + delta * rate);\n" +
            "\n" +
            "-- 判断是否有足够令牌\n" +
            "if new_tokens >= 1 then\n" +
            "    new_tokens = new_tokens - 1;\n" +
            "    redis.call('HMSET', key, 'tokens', new_tokens, 'last_time', now);\n" +
            "    redis.call('EXPIRE', key, 60);\n" +
            "    return 1;\n" +
            "else\n" +
            "    redis.call('HSET', key, 'last_time', now);\n" +
            "    return 0;\n" +
            "end;";
        
        Long result = redisTemplate.execute(
            new DefaultRedisScript<>(luaScript, Long.class),
            Collections.singletonList(redisKey),
            String.valueOf(capacity),
            String.valueOf(rate),
            String.valueOf(now)
        );
        
        return result != null && result == 1;
    }
}
```

## 三、Spring Boot限流注解实现

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RateLimit {
    String key() default "";           // 限流标识
    int limit() default 100;           // 限制次数
    int window() default 60;           // 窗口大小（秒）
    LimitType type() default LimitType.FIXED;  // 限流类型
}

public enum LimitType {
    FIXED,      // 固定窗口
    SLIDING,    // 滑动窗口
    TOKEN       // 令牌桶
}

@Aspect
@Component
public class RateLimitAspect {
    
    @Autowired
    private FixedWindowRateLimiter fixedLimiter;
    @Autowired
    private SlidingWindowRateLimiter slidingLimiter;
    @Autowired
    private TokenBucketRateLimiter tokenLimiter;
    
    @Around("@annotation(rateLimit)")
    public Object around(ProceedingJoinPoint point, RateLimit rateLimit) throws Throwable {
        String key = generateKey(point, rateLimit);
        
        boolean allowed = switch (rateLimit.type()) {
            case FIXED -> fixedLimiter.isAllowed(key, rateLimit.limit(), rateLimit.window());
            case SLIDING -> slidingLimiter.isAllowed(key, rateLimit.limit(), rateLimit.window());
            case TOKEN -> tokenLimiter.isAllowed(key, rateLimit.limit(), rateLimit.limit() / (double) rateLimit.window());
        };
        
        if (!allowed) {
            throw new RateLimitException("请求过于频繁，请稍后再试");
        }
        
        return point.proceed();
    }
    
    private String generateKey(ProceedingJoinPoint point, RateLimit rateLimit) {
        if (!rateLimit.key().isEmpty()) {
            return rateLimit.key();
        }
        // 默认使用类名+方法名
        MethodSignature signature = (MethodSignature) point.getSignature();
        return signature.getDeclaringTypeName() + ":" + signature.getName();
    }
}

// 使用示例
@RestController
public class ApiController {
    
    @RateLimit(key = "api:user:list", limit = 100, window = 60, type = LimitType.SLIDING)
    @GetMapping("/api/users")
    public List<User> listUsers() {
        return userService.list();
    }
    
    @RateLimit(key = "api:order:create", limit = 10, window = 60, type = LimitType.TOKEN)
    @PostMapping("/api/orders")
    public Order createOrder(@RequestBody OrderRequest request) {
        return orderService.create(request);
    }
}
```

## 四、实际应用场景

| 场景 | 限流策略 | 实现方式 |
|------|----------|----------|
| **API网关限流** | 令牌桶，1000/秒 | 网关层统一拦截 |
| **用户操作限流** | 滑动窗口，5/分钟 | 基于用户ID限流 |
| **短信发送** | 固定窗口，1/小时 | 基于手机号限流 |
| **登录尝试** | 滑动窗口，5/15分钟 | 基于IP+用户名限流 |
| **库存扣减** | 令牌桶，突发100 | 秒杀场景保护 |

**使用场景：**
- **API限流**：保护后端服务，防止被刷接口
- **用户操作频率限制**：防止恶意操作、暴力破解
- **资源保护**：数据库连接池、线程池等资源保护
- **公平使用**：确保每个用户都能公平使用资源

---

### 22. Redis如何实现消息队列？

**答案：**

## 一、三种方案对比

| 特性 | List | Pub/Sub | Stream |
|------|------|---------|--------|
| **消息持久化** | ✅ 是 | ❌ 否 | ✅ 是 |
| **消费者确认** | ❌ 否 | ❌ 否 | ✅ ACK机制 |
| **消费者组** | ❌ 否 | ❌ 否 | ✅ 支持 |
| **消息回溯** | ❌ 否 | ❌ 否 | ✅ 支持 |
| **阻塞消费** | ✅ BRPOP | ✅ 原生 | ✅ XREAD |
| **适用场景** | 简单队列 | 实时通知 | 可靠消息系统 |

## 二、List实现简单队列

**基本原理：**
- 使用LPUSH入队，RPOP/BRPOP出队，实现FIFO队列
- BRPOP是阻塞版本，没有消息时等待，避免空轮询

```bash
# 生产者入队
LPUSH queue:order "{orderId:1001, userId:123, amount:99.9}"
LPUSH queue:order "{orderId:1002, userId:456, amount:199.9}"

# 消费者阻塞出队（最多等待30秒）
BRPOP queue:order 30

# 获取队列长度
LLEN queue:order
```

**Java实现：**

```java
@Service
public class ListQueueService {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    private static final String QUEUE_KEY = "queue:order";
    
    /**
     * 发送消息
     */
    public void sendMessage(String message) {
        redisTemplate.opsForList().leftPush(QUEUE_KEY, message);
    }
    
    /**
     * 批量发送
     */
    public void sendMessages(List<String> messages) {
        redisTemplate.executePipelined((RedisCallback<Object>) connection -> {
            for (String message : messages) {
                connection.listCommands().lPush(
                    QUEUE_KEY.getBytes(),
                    message.getBytes()
                );
            }
            return null;
        });
    }
    
    /**
     * 阻塞消费（单消费者）
     */
    public void consumeBlocking() {
        while (!Thread.currentThread().isInterrupted()) {
            try {
                // 阻塞等待，超时时间60秒
                String message = redisTemplate.opsForList()
                    .rightPop(QUEUE_KEY, 60, TimeUnit.SECONDS);
                
                if (message != null) {
                    processMessage(message);
                }
            } catch (Exception e) {
                log.error("消费消息失败", e);
            }
        }
    }
    
    /**
     * 多消费者竞争消费
     */
    @Async("taskExecutor")
    public void consumeCompeting(String consumerId) {
        while (true) {
            String message = redisTemplate.opsForList()
                .rightPop(QUEUE_KEY, 5, TimeUnit.SECONDS);
            
            if (message != null) {
                log.info("消费者{}处理消息: {}", consumerId, message);
                processMessage(message);
            }
        }
    }
    
    private void processMessage(String message) {
        // 业务处理逻辑
        try {
            OrderMessage order = JSON.parseObject(message, OrderMessage.class);
            orderService.processOrder(order);
        } catch (Exception e) {
            // 失败处理：可以放入死信队列
            redisTemplate.opsForList().leftPush("queue:order:dlq", message);
        }
    }
}
```

**List队列的局限性：**
- 没有ACK机制，消费者处理失败消息丢失
- 不支持多播（一个消息只能被一个消费者处理）
- 无法查看未消费消息数量

## 三、Pub/Sub发布订阅

**基本原理：**
- 发布者将消息发送到频道，所有订阅者同时接收
- 消息不存储，实时推送给在线订阅者

```bash
# 订阅者1订阅频道
SUBSCRIBE channel:order channel:payment

# 订阅者2订阅频道（支持通配符）
PSUBSCRIBE channel:*

# 发布者发送消息
PUBLISH channel:order "新订单通知"

# 查看频道订阅数
PUBSUB NUMSUB channel:order
```

**Java实现：**

```java
@Service
public class PubSubService {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    /**
     * 发布消息
     */
    public void publish(String channel, Object message) {
        String jsonMessage = JSON.toJSONString(message);
        redisTemplate.convertAndSend(channel, jsonMessage);
    }
    
    /**
     * 订阅消息（需要在配置中设置MessageListener）
     */
    @Bean
    public RedisMessageListenerContainer container(
            RedisConnectionFactory connectionFactory) {
        
        RedisMessageListenerContainer container = 
            new RedisMessageListenerContainer();
        container.setConnectionFactory(connectionFactory);
        
        // 订阅订单频道
        container.addMessageListener(
            orderListener(),
            new PatternTopic("channel:order:*")
        );
        
        return container;
    }
    
    @Bean
    public MessageListener orderListener() {
        return (message, pattern) -> {
            String body = new String(message.getBody());
            String channel = new String(message.getChannel());
            
            log.info("收到频道{}的消息: {}", channel, body);
            
            // 根据频道处理不同类型的消息
            if (channel.contains("created")) {
                handleOrderCreated(body);
            } else if (channel.contains("paid")) {
                handleOrderPaid(body);
            }
        };
    }
}
```

**Pub/Sub适用场景：**
- 实时通知（如WebSocket推送）
- 配置变更广播
- 缓存失效通知

**Pub/Sub的局限性：**
- 消息不持久化，消费者离线期间消息丢失
- 没有ACK机制，无法保证消息必达
- 发布者不知道有多少订阅者收到消息

## 四、Stream实现可靠消息队列（推荐）

**Stream核心概念：**
- **消息ID**：时间戳-序列号，保证全局唯一且递增
- **消费者组**：多个消费者共同消费，消息只会被组内一个消费者处理
- **ACK机制**：消费者处理完成后确认，未确认消息可被重新消费
- **Pending列表**：记录已投递但未ACK的消息

```bash
# 生产者添加消息（*表示自动生成ID）
XADD order_stream * orderId 1001 userId 123 status created

# 创建消费者组（从最新消息开始消费）
XGROUP CREATE order_stream order_group $

# 消费者读取消息（>表示读取新消息）
XREADGROUP GROUP order_group consumer1 COUNT 1 STREAMS order_stream >

# 确认消息已处理
XACK order_stream order_group 1700000000000-0

# 查看Pending列表（未确认消息）
XPENDING order_stream order_group - + 10

# 消费者故障转移后，认领超时未确认的消息
XCLAIM order_stream order_group consumer2 60000 1700000000000-0
```

**Java完整实现：**

```java
@Service
public class StreamQueueService {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    private static final String STREAM_KEY = "stream:order";
    private static final String GROUP_NAME = "order_group";
    
    /**
     * 初始化消费者组
     */
    @PostConstruct
    public void init() {
        try {
            redisTemplate.opsForStream()
                .createGroup(STREAM_KEY, ReadOffset.latest(), GROUP_NAME);
        } catch (RedisSystemException e) {
            // 消费者组已存在，忽略错误
            log.info("消费者组已存在: {}", GROUP_NAME);
        }
    }
    
    /**
     * 发送消息
     */
    public String sendMessage(Map<String, Object> message) {
        RecordId recordId = redisTemplate.opsForStream()
            .add(StreamRecords.newRecord()
                .in(STREAM_KEY)
                .ofMap(message));
        return recordId.getValue();
    }
    
    /**
     * 发送订单消息
     */
    public String sendOrder(Order order) {
        Map<String, Object> message = new HashMap<>();
        message.put("orderId", order.getOrderId());
        message.put("userId", order.getUserId());
        message.put("amount", order.getAmount());
        message.put("status", order.getStatus());
        message.put("createTime", System.currentTimeMillis());
        
        return sendMessage(message);
    }
    
    /**
     * 消费者（支持ACK和故障转移）
     */
    @Async("streamConsumerExecutor")
    public void consume(String consumerName) {
        while (!Thread.currentThread().isInterrupted()) {
            try {
                // 读取消息，阻塞5秒
                List<MapRecord<String, Object, Object>> records = 
                    redisTemplate.opsForStream().read(
                        Consumer.from(GROUP_NAME, consumerName),
                        StreamReadOptions.empty()
                            .count(10)
                            .block(Duration.ofSeconds(5)),
                        StreamOffset.create(STREAM_KEY, ReadOffset.lastConsumed())
                    );
                
                if (records == null || records.isEmpty()) {
                    continue;
                }
                
                for (MapRecord<String, Object, Object> record : records) {
                    try {
                        processRecord(record);
                        // 确认消息
                        redisTemplate.opsForStream()
                            .acknowledge(STREAM_KEY, GROUP_NAME, record.getId());
                    } catch (Exception e) {
                        log.error("处理消息失败: {}", record.getId(), e);
                        // 不ACK，消息会进入Pending列表，稍后重试
                    }
                }
            } catch (Exception e) {
                log.error("消费异常", e);
            }
        }
    }
    
    /**
     * 处理Pending列表中的超时消息（故障转移）
     */
    @Scheduled(fixedDelay = 30000)  // 每30秒执行一次
    public void processPendingMessages() {
        try {
            // 获取Pending列表中的消息（已超时30秒未确认）
            PendingMessagesSummary summary = redisTemplate.opsForStream()
                .pending(STREAM_KEY, GROUP_NAME);
            
            if (summary.getTotalPendingMessages() == 0) {
                return;
            }
            
            // 获取具体的消息ID
            PendingMessages messages = redisTemplate.opsForStream()
                .pending(STREAM_KEY, Consumer.from(GROUP_NAME, "consumer1"),
                    Range.unbounded(), 100);
            
            for (PendingMessage message : messages) {
                // 超过30秒未确认，重新投递
                if (message.getElapsedTimeSinceLastDelivery().getSeconds() > 30) {
                    redisTemplate.opsForStream().claim(
                        STREAM_KEY, GROUP_NAME, "consumer_backup",
                        Duration.ofSeconds(30), message.getId()
                    );
                    log.info("认领超时消息: {}", message.getId());
                }
            }
        } catch (Exception e) {
            log.error("处理Pending消息失败", e);
        }
    }
    
    private void processRecord(MapRecord<String, Object, Object> record) {
        Map<Object, Object> value = record.getValue();
        Order order = new Order();
        order.setOrderId((String) value.get("orderId"));
        order.setUserId((String) value.get("userId"));
        order.setAmount(new BigDecimal((String) value.get("amount")));
        
        log.info("消费者处理订单: {}", order.getOrderId());
        orderService.processOrder(order);
    }
}
```

**Stream高级特性：**

```java
@Service
public class StreamAdvancedService {
    
    /**
     * 消息回溯（从指定ID开始消费）
     */
    public void replayFromId(String startId) {
        List<MapRecord<String, Object, Object>> records = 
            redisTemplate.opsForStream().read(
                StreamReadOptions.empty().count(100),
                StreamOffset.create("stream:order", ReadOffset.from(startId))
            );
        
        for (MapRecord<String, Object, Object> record : records) {
            log.info("回溯消息: {} -> {}", record.getId(), record.getValue());
        }
    }
    
    /**
     * 删除已处理的消息（控制Stream长度）
     */
    public void trimStream() {
        // 保留最近10000条消息
        redisTemplate.opsForStream()
            .trim("stream:order", 10000, true);
    }
    
    /**
     * 监控Stream状态
     */
    public StreamInfo.XInfoStream getStreamInfo() {
        return redisTemplate.opsForStream()
            .info("stream:order");
    }
}
```

## 五、方案选型建议

| 场景 | 推荐方案 | 理由 |
|------|----------|------|
| **简单任务队列** | List | 实现简单，满足基本需求 |
| **实时通知/广播** | Pub/Sub | 低延迟，支持多播 |
| **可靠消息处理** | Stream | 支持ACK、消费者组、故障转移 |
| **日志收集** | Stream | 支持消息回溯，可持久化存储 |
| **延迟队列** | Stream + 定时任务 | 利用消息ID的时间戳特性 |

**使用场景：**
- **异步任务处理**：订单处理、邮件发送、数据同步
- **实时通知系统**：在线状态推送、消息提醒
- **日志收集**：应用日志、操作审计
- **事件驱动架构**：微服务间的事件通知

---

### 23. Redis如何实现分布式会话（Session）？

**答案：**

## 一、为什么需要分布式Session？

**传统Session的问题：**
- Session存储在应用服务器内存中
- 多台服务器之间Session无法共享
- 用户请求被负载均衡到不同服务器时，Session丢失

**解决方案对比：**

| 方案 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| **Session复制** | 实现简单 | 占用带宽，扩展性差 | 小规模集群 |
| **Session粘滞** | 无额外依赖 | 负载不均，单点故障 | 简单场景 |
| **数据库存储** | 持久化 | 性能差，IO压力大 | 不推荐 |
| **Redis存储** | 高性能，易扩展 | 需要额外维护 | 大规模集群 |

## 二、Spring Session + Redis实现

**Maven依赖：**

```xml
<dependencies>
    <!-- Spring Session Redis -->
    <dependency>
        <groupId>org.springframework.session</groupId>
        <artifactId>spring-session-data-redis</artifactId>
    </dependency>
    
    <!-- Redis连接池 -->
    <dependency>
        <groupId>io.lettuce</groupId>
        <artifactId>lettuce-core</artifactId>
    </dependency>
</dependencies>
```

**基础配置：**

```java
@Configuration
@EnableRedisHttpSession(
    maxInactiveIntervalInSeconds = 1800,  // Session过期时间30分钟
    redisNamespace = "myapp:session"        // Redis key前缀
)
public class SessionConfig {
    
    @Bean
    public LettuceConnectionFactory connectionFactory() {
        RedisStandaloneConfiguration config = 
            new RedisStandaloneConfiguration("localhost", 6379);
        config.setPassword(RedisPassword.of("password"));
        
        return new LettuceConnectionFactory(config);
    }
    
    /**
     * 自定义Cookie配置
     */
    @Bean
    public CookieSerializer cookieSerializer() {
        DefaultCookieSerializer serializer = new DefaultCookieSerializer();
        serializer.setCookieName("MYAPP_SESSION");  // Cookie名称
        serializer.setCookiePath("/");
        serializer.setDomainName("example.com");     // 跨子域共享
        serializer.setUseHttpOnlyCookie(true);       // 防止XSS
        serializer.setUseSecureCookie(true);         // 仅HTTPS传输
        serializer.setSameSite("Strict");            // CSRF防护
        return serializer;
    }
}
```

**Session数据结构详解：**

```bash
# Spring Session在Redis中的存储结构

# 1. Session主数据（Hash类型）
HSET spring:session:sessions:abc123 
    creationTime "1700000000000"
    lastAccessedTime "1700000100000"
    maxInactiveInterval "1800"
    sessionAttr:username "zhangsan"
    sessionAttr:userId "10001"
    sessionAttr:roles "[ADMIN,USER]"

# 2. Session过期时间（ZSet类型，用于定时清理）
ZADD spring:session:expirations:1700001900 "abc123"

# 3. 用户Session索引（用于单用户多设备管理）
SADD spring:session:index:user:10001 "abc123"
```

## 三、自定义Session管理

**自定义Redis Session Repository：**

```java
@Service
public class CustomSessionService {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    private static final String SESSION_PREFIX = "session:user:";
    private static final long SESSION_TIMEOUT = 30 * 60; // 30分钟
    
    /**
     * 创建Session
     */
    public String createSession(User user) {
        String sessionId = UUID.randomUUID().toString().replace("-", "");
        String key = SESSION_PREFIX + sessionId;
        
        Map<String, String> sessionData = new HashMap<>();
        sessionData.put("userId", user.getId());
        sessionData.put("username", user.getUsername());
        sessionData.put("roles", JSON.toJSONString(user.getRoles()));
        sessionData.put("loginTime", String.valueOf(System.currentTimeMillis()));
        sessionData.put("ip", getClientIp());
        
        // 存储到Redis
        redisTemplate.opsForHash().putAll(key, sessionData);
        redisTemplate.expire(key, SESSION_TIMEOUT, TimeUnit.SECONDS);
        
        // 记录用户Session索引（支持单用户多设备）
        redisTemplate.opsForSet().add("user:sessions:" + user.getId(), sessionId);
        
        return sessionId;
    }
    
    /**
     * 获取Session
     */
    public UserSession getSession(String sessionId) {
        String key = SESSION_PREFIX + sessionId;
        Map<Object, Object> entries = redisTemplate.opsForHash().entries(key);
        
        if (entries.isEmpty()) {
            return null;
        }
        
        // 刷新过期时间
        redisTemplate.expire(key, SESSION_TIMEOUT, TimeUnit.SECONDS);
        
        return convertToUserSession(entries);
    }
    
    /**
     * 销毁Session（登出）
     */
    public void destroySession(String sessionId) {
        String key = SESSION_PREFIX + sessionId;
        
        // 获取用户ID
        String userId = (String) redisTemplate.opsForHash().get(key, "userId");
        
        // 删除Session
        redisTemplate.delete(key);
        
        // 从用户索引中移除
        if (userId != null) {
            redisTemplate.opsForSet().remove("user:sessions:" + userId, sessionId);
        }
    }
    
    /**
     * 踢出用户所有Session（强制下线）
     */
    public void kickoutUser(String userId) {
        Set<String> sessionIds = redisTemplate.opsForSet()
            .members("user:sessions:" + userId);
        
        if (sessionIds != null) {
            for (String sessionId : sessionIds) {
                redisTemplate.delete(SESSION_PREFIX + sessionId);
            }
            redisTemplate.delete("user:sessions:" + userId);
        }
    }
    
    /**
     * 获取用户在线设备数
     */
    public Long getUserDeviceCount(String userId) {
        return redisTemplate.opsForSet()
            .size("user:sessions:" + userId);
    }
}
```

## 四、Session安全性增强

**1. Session防劫持：**

```java
@Component
public class SessionSecurityFilter extends OncePerRequestFilter {
    
    @Autowired
    private CustomSessionService sessionService;
    
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        
        String sessionId = extractSessionId(request);
        if (sessionId != null) {
            UserSession session = sessionService.getSession(sessionId);
            
            if (session != null) {
                // 校验IP是否变化（可选，根据安全需求）
                String currentIp = getClientIp(request);
                if (!currentIp.equals(session.getIp())) {
                    // IP变化，可能是Session劫持
                    log.warn("Session IP不匹配，可能是劫持: {}", sessionId);
                    // 可以选择：1.拒绝请求 2.重新验证 3.记录日志
                }
                
                // 校验User-Agent
                String currentUa = request.getHeader("User-Agent");
                if (!currentUa.equals(session.getUserAgent())) {
                    log.warn("Session User-Agent不匹配: {}", sessionId);
                }
                
                // 将Session绑定到请求上下文
                RequestContext.setCurrentUser(session);
            }
        }
        
        chain.doFilter(request, response);
    }
}
```

**2. Session序列化方案对比：**

| 序列化方式 | 优点 | 缺点 | 适用场景 |
|------------|------|------|----------|
| **JDK** | 无需依赖 | 体积大，速度慢 | 简单场景 |
| **JSON** | 可读性好，跨语言 | 类型信息丢失 | 推荐 |
| **Kryo** | 体积小，速度快 | 需要注册类 | 高性能场景 |
| **Protobuf** | 体积最小 | 需要定义Schema | 微服务间通信 |

**自定义Kryo序列化：**

```java
public class KryoSerializer implements RedisSerializer<Object> {
    
    private static final ThreadLocal<Kryo> kryoPool = ThreadLocal.withInitial(() -> {
        Kryo kryo = new Kryo();
        kryo.register(UserSession.class);
        kryo.register(ArrayList.class);
        kryo.register(HashMap.class);
        return kryo;
    });
    
    @Override
    public byte[] serialize(Object object) throws SerializationException {
        if (object == null) {
            return new byte[0];
        }
        
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        Output output = new Output(baos);
        kryoPool.get().writeClassAndObject(output, object);
        output.close();
        return baos.toByteArray();
    }
    
    @Override
    public Object deserialize(byte[] bytes) throws SerializationException {
        if (bytes == null || bytes.length == 0) {
            return null;
        }
        
        Input input = new Input(bytes);
        Object object = kryoPool.get().readClassAndObject(input);
        input.close();
        return object;
    }
}
```

## 五、Session共享的高级场景

**1. 单点登录（SSO）实现：**

```java
@Service
public class SsoSessionService {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    private static final String SSO_TOKEN_PREFIX = "sso:token:";
    private static final String SSO_USER_PREFIX = "sso:user:";
    
    /**
     * 生成SSO Token
     */
    public String generateSsoToken(User user) {
        String token = UUID.randomUUID().toString();
        
        // Token -> 用户信息
        redisTemplate.opsForValue().set(
            SSO_TOKEN_PREFIX + token,
            JSON.toJSONString(user),
            30, TimeUnit.MINUTES
        );
        
        // 用户 -> Token列表（支持多系统登录）
        redisTemplate.opsForSet().add(SSO_USER_PREFIX + user.getId(), token);
        
        return token;
    }
    
    /**
     * 验证SSO Token
     */
    public User validateSsoToken(String token) {
        String json = redisTemplate.opsForValue().get(SSO_TOKEN_PREFIX + token);
        if (json == null) {
            return null;
        }
        
        // 刷新Token过期时间
        redisTemplate.expire(SSO_TOKEN_PREFIX + token, 30, TimeUnit.MINUTES);
        
        return JSON.parseObject(json, User.class);
    }
    
    /**
     * 全局登出
     */
    public void globalLogout(String userId) {
        Set<String> tokens = redisTemplate.opsForSet()
            .members(SSO_USER_PREFIX + userId);
        
        if (tokens != null) {
            for (String token : tokens) {
                redisTemplate.delete(SSO_TOKEN_PREFIX + token);
            }
        }
        
        redisTemplate.delete(SSO_USER_PREFIX + userId);
    }
}
```

**2. Session监控与管理：**

```java
@RestController
@RequestMapping("/admin/session")
public class SessionAdminController {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    /**
     * 获取在线用户统计
     */
    @GetMapping("/stats")
    public SessionStats getSessionStats() {
        // 统计Session数量
        Set<String> sessionKeys = redisTemplate.keys("session:user:*");
        
        // 统计活跃用户（最近5分钟有操作）
        long activeCount = sessionKeys != null ? sessionKeys.stream()
            .filter(key -> {
                Long ttl = redisTemplate.getExpire(key);
                return ttl != null && ttl > 25 * 60; // 剩余TTL大于25分钟
            }).count() : 0;
        
        return SessionStats.builder()
            .totalSessions(sessionKeys != null ? sessionKeys.size() : 0)
            .activeUsers((int) activeCount)
            .build();
    }
    
    /**
     * 强制用户下线
     */
    @PostMapping("/kickout/{userId}")
    public Result kickoutUser(@PathVariable String userId) {
        customSessionService.kickoutUser(userId);
        return Result.success();
    }
}
```

**使用场景：**
- **集群环境Session共享**：多台Tomcat/Nginx负载均衡
- **微服务Session共享**：网关统一认证，服务间免登录
- **单点登录（SSO）**：多系统统一认证中心
- **实时在线用户管理**：查看在线用户、强制下线

---

### 24. Redis如何实现附近的人功能？

**答案：**

## 一、GeoHash原理

**为什么需要GeoHash？**
- 二维坐标（经度、纬度）无法直接用Sorted Set排序
- GeoHash将二维坐标编码为一维字符串，支持范围查询

**GeoHash编码过程：**

```
1. 经度范围[-180, 180]，纬度范围[-90, 90]
2. 采用二分法，每次将区间分为两半
3. 在右半区记为1，左半区记为0
4. 经度和纬度交替编码，最终得到二进制串
5. 每5位一组转换为Base32编码

示例：北京天安门(116.3974, 39.9092)
经度编码：11010 01101 01001 10111 10101
纬度编码：10101 10001 11001 11000 10110
交替组合：11100 11001 01100 01111 11100 01101
Base32：  wx4g 0ec8
```

**GeoHash特性：**
- 字符串前缀相同的点距离相近
- 但距离近的点不一定前缀相同（边界问题）
- 精度可控：字符越长，精度越高

| GeoHash长度 | 精度（km） | 适用场景 |
|-------------|------------|----------|
| 4 | ±20 | 城市级别 |
| 5 | ±2.4 | 区域级别 |
| 6 | ±0.61 | 街道级别 |
| 7 | ±0.076 | 建筑级别 |
| 8 | ±0.019 | 精确位置 |

## 二、Redis Geo基础命令

```bash
# 添加位置（支持批量添加）
GEOADD locations 116.3974 39.9092 "user1" 116.4074 39.9192 "user2"

# 查询附近的人（以指定坐标为中心）
GEORADIUS locations 116.3974 39.9092 5 km WITHDIST WITHCOORD COUNT 10 ASC

# 查询附近的人（以指定成员为中心）
GEORADIUSBYMEMBER locations "user1" 5 km WITHDIST

# 计算两个成员之间的距离
GEODIST locations "user1" "user2" km

# 获取成员的坐标
GEOPOS locations "user1"

# 获取成员的GeoHash编码
GEOHASH locations "user1"

# 删除成员
ZREM locations "user1"
```

**命令参数说明：**
- `WITHDIST`：返回距离
- `WITHCOORD`：返回坐标
- `WITHHASH`：返回GeoHash编码
- `COUNT n`：限制返回数量
- `ASC/DESC`：按距离排序

## 三、Java完整实现

```java
@Service
public class GeoLocationService {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    private static final String GEO_KEY = "geo:users";
    private static final String USER_LOC_KEY = "user:location:";
    
    /**
     * 更新用户位置
     */
    public void updateLocation(String userId, double longitude, double latitude) {
        // 添加到Geo集合
        redisTemplate.opsForGeo().add(GEO_KEY, 
            new Point(longitude, latitude), userId);
        
        // 记录用户最后位置（用于快速查询）
        String key = USER_LOC_KEY + userId;
        Map<String, String> location = new HashMap<>();
        location.put("longitude", String.valueOf(longitude));
        location.put("latitude", String.valueOf(latitude));
        location.put("updateTime", String.valueOf(System.currentTimeMillis()));
        
        redisTemplate.opsForHash().putAll(key, location);
        redisTemplate.expire(key, 7, TimeUnit.DAYS);  // 7天过期
    }
    
    /**
     * 查询附近的人
     */
    public List<NearbyUser> findNearbyUsers(String userId, double radius, 
                                             int count) {
        // 获取用户当前位置
        Point center = getUserLocation(userId);
        if (center == null) {
            return Collections.emptyList();
        }
        
        // 查询附近用户
        GeoResults<RedisGeoCommands.GeoLocation<String>> results = 
            redisTemplate.opsForGeo().radius(GEO_KEY,
                new Circle(center, new Distance(radius, Metrics.KILOMETERS)),
                RedisGeoCommands.GeoRadiusCommandArgs.newGeoRadiusArgs()
                    .includeDistance()
                    .includeCoordinates()
                    .sortAscending()
                    .limit(count)
            );
        
        return convertToNearbyUsers(results, userId);
    }
    
    /**
     * 查询指定坐标附近的人
     */
    public List<NearbyUser> findNearbyByCoordinate(double longitude, 
                                                    double latitude, 
                                                    double radius, 
                                                    int count) {
        GeoResults<RedisGeoCommands.GeoLocation<String>> results = 
            redisTemplate.opsForGeo().radius(GEO_KEY,
                new Circle(new Point(longitude, latitude), 
                          new Distance(radius, Metrics.KILOMETERS)),
                RedisGeoCommands.GeoRadiusCommandArgs.newGeoRadiusArgs()
                    .includeDistance()
                    .includeCoordinates()
                    .sortAscending()
                    .limit(count)
            );
        
        return convertToNearbyUsers(results, null);
    }
    
    /**
     * 计算两个用户之间的距离
     */
    public Double getDistance(String userId1, String userId2) {
        Distance distance = redisTemplate.opsForGeo()
            .distance(GEO_KEY, userId1, userId2, Metrics.KILOMETERS);
        return distance != null ? distance.getValue() : null;
    }
    
    /**
     * 批量导入位置数据（Pipeline优化）
     */
    public void batchImportLocations(Map<String, Point> userLocations) {
        redisTemplate.executePipelined((RedisCallback<Object>) connection -> {
            for (Map.Entry<String, Point> entry : userLocations.entrySet()) {
                connection.geoCommands().geoAdd(
                    GEO_KEY.getBytes(),
                    entry.getValue(),
                    entry.getKey().getBytes()
                );
            }
            return null;
        });
    }
    
    /**
     * 移除用户位置
     */
    public void removeLocation(String userId) {
        redisTemplate.opsForGeo().remove(GEO_KEY, userId);
        redisTemplate.delete(USER_LOC_KEY + userId);
    }
    
    private Point getUserLocation(String userId) {
        String key = USER_LOC_KEY + userId;
        Map<Object, Object> entries = redisTemplate.opsForHash().entries(key);
        
        if (entries.isEmpty()) {
            return null;
        }
        
        double longitude = Double.parseDouble((String) entries.get("longitude"));
        double latitude = Double.parseDouble((String) entries.get("latitude"));
        
        return new Point(longitude, latitude);
    }
    
    private List<NearbyUser> convertToNearbyUsers(
            GeoResults<RedisGeoCommands.GeoLocation<String>> results,
            String excludeUserId) {
        
        if (results == null) {
            return Collections.emptyList();
        }
        
        List<NearbyUser> users = new ArrayList<>();
        for (GeoResult<RedisGeoCommands.GeoLocation<String>> result : results) {
            String userId = result.getContent().getName();
            
            // 排除自己
            if (userId.equals(excludeUserId)) {
                continue;
            }
            
            NearbyUser user = new NearbyUser();
            user.setUserId(userId);
            user.setDistance(result.getDistance().getValue());
            
            Point point = result.getContent().getPoint();
            if (point != null) {
                user.setLongitude(point.getX());
                user.setLatitude(point.getY());
            }
            
            users.add(user);
        }
        
        return users;
    }
}

// 数据类
@Data
public class NearbyUser {
    private String userId;
    private Double distance;  // 距离（公里）
    private Double longitude;
    private Double latitude;
}
```

## 四、高级功能实现

**1. 附近商家搜索（带筛选条件）：**

```java
@Service
public class NearbyShopService {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    /**
     * 搜索附近商家（支持分类筛选）
     */
    public List<NearbyShop> searchNearbyShops(double longitude, double latitude,
                                               double radius, String category,
                                               int minRating) {
        // 1. 先按距离查询
        List<NearbyUser> nearbyUsers = geoService.findNearbyByCoordinate(
            longitude, latitude, radius, 100);
        
        // 2. 获取商家详细信息并筛选
        List<NearbyShop> shops = new ArrayList<>();
        for (NearbyUser user : nearbyUsers) {
            // 从Hash获取商家详情
            String shopKey = "shop:info:" + user.getUserId();
            Map<Object, Object> shopInfo = redisTemplate.opsForHash().entries(shopKey);
            
            if (shopInfo.isEmpty()) {
                continue;
            }
            
            // 分类筛选
            String shopCategory = (String) shopInfo.get("category");
            if (category != null && !category.equals(shopCategory)) {
                continue;
            }
            
            // 评分筛选
            int rating = Integer.parseInt((String) shopInfo.get("rating"));
            if (rating < minRating) {
                continue;
            }
            
            NearbyShop shop = new NearbyShop();
            shop.setShopId(user.getUserId());
            shop.setName((String) shopInfo.get("name"));
            shop.setDistance(user.getDistance());
            shop.setRating(rating);
            shop.setCategory(shopCategory);
            
            shops.add(shop);
        }
        
        // 3. 按评分排序
        shops.sort((a, b) -> b.getRating() - a.getRating());
        
        return shops;
    }
}
```

**2. 位置数据分片（大数据量优化）：**

```java
@Service
public class ShardedGeoService {
    
    /**
     * 根据城市分片存储位置数据
     */
    public void updateLocation(String cityCode, String userId, 
                               double longitude, double latitude) {
        String geoKey = "geo:city:" + cityCode;
        redisTemplate.opsForGeo().add(geoKey, 
            new Point(longitude, latitude), userId);
    }
    
    /**
     * 查询附近的人（先确定城市，再查询）
     */
    public List<NearbyUser> findNearby(String cityCode, double longitude,
                                        double latitude, double radius) {
        String geoKey = "geo:city:" + cityCode;
        
        GeoResults<RedisGeoCommands.GeoLocation<String>> results = 
            redisTemplate.opsForGeo().radius(geoKey,
                new Circle(new Point(longitude, latitude), 
                          new Distance(radius, Metrics.KILOMETERS)),
                RedisGeoCommands.GeoRadiusCommandArgs.newGeoRadiusArgs()
                    .includeDistance()
                    .sortAscending()
                    .limit(50)
            );
        
        return convertToNearbyUsers(results, null);
    }
    
    /**
     * 根据坐标确定城市（简化示例）
     */
    public String getCityCode(double longitude, double latitude) {
        // 实际应用中可以使用GeoHash前缀匹配或反向地理编码
        // 这里简化处理
        if (longitude > 115 && longitude < 118 && 
            latitude > 39 && latitude < 41) {
            return "110000";  // 北京
        }
        return "default";
    }
}
```

**3. 位置数据过期清理：**

```java
@Component
public class GeoCleanupTask {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    /**
     * 清理不活跃用户的地理位置（每天凌晨执行）
     */
    @Scheduled(cron = "0 0 2 * * ?")
    public void cleanupInactiveUsers() {
        // 获取所有用户位置
        Set<String> userKeys = redisTemplate.keys("user:location:*");
        
        if (userKeys == null) {
            return;
        }
        
        long now = System.currentTimeMillis();
        long inactiveThreshold = 7 * 24 * 60 * 60 * 1000;  // 7天
        
        for (String key : userKeys) {
            String updateTime = (String) redisTemplate.opsForHash()
                .get(key, "updateTime");
            
            if (updateTime != null) {
                long lastUpdate = Long.parseLong(updateTime);
                if (now - lastUpdate > inactiveThreshold) {
                    String userId = key.replace("user:location:", "");
                    // 从Geo集合中移除
                    redisTemplate.opsForGeo().remove("geo:users", userId);
                    // 删除位置记录
                    redisTemplate.delete(key);
                    
                    log.info("清理不活跃用户位置: {}", userId);
                }
            }
        }
    }
}
```

## 五、性能优化建议

| 优化点 | 方案 | 效果 |
|--------|------|------|
| **大数据量** | 按城市/区域分片 | 降低单个ZSet大小 |
| **高并发写入** | Pipeline批量添加 | 减少网络RTT |
| **查询优化** | 限制COUNT，避免大半径查询 | 减少计算量 |
| **内存优化** | 定期清理不活跃用户 | 节省内存空间 |
| **冷热分离** | 活跃用户在Redis，历史数据归档 | 平衡性能和成本 |

**使用场景：**
- **附近的人**：社交应用查找附近用户
- **附近商家**：O2O应用搜索周边店铺
- **位置服务**：外卖配送、网约车调度
- **地理围栏**：电子围栏、区域监控

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
