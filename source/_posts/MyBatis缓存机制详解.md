---
title: MyBatis缓存机制详解：从一级缓存到二级缓存，全面掌握性能优化利器
date: 2026-04-20 14:00:00
tags:
  - MyBatis
  - Java
  - 缓存
  - 性能优化
categories:
  - 后端开发
description: 深入剖析MyBatis一级缓存和二级缓存的工作原理、配置方式、源码实现以及最佳实践，帮助开发者掌握缓存机制提升应用性能。
---

# MyBatis缓存机制详解：从一级缓存到二级缓存，全面掌握性能优化利器

MyBatis 提供了强大而灵活的缓存机制，包括一级缓存和二级缓存两个层次，旨在减少数据库访问次数，提高应用性能。

## 1. 一级缓存（SqlSession 级别缓存）

一级缓存是 MyBatis 最基本的缓存机制，也称为本地缓存或会话缓存。

### 工作原理

- 一级缓存默认开启，无法关闭
- 缓存作用域为 SqlSession，即同一个 SqlSession 中执行的相同查询会使用缓存
- 缓存存储在每个 SqlSession 对象中的 LocalCache(PerpetualCache)实例中
- 缓存的 key 由 SQL 语句、参数、分页等信息组成，确保相同查询得到相同结果
- 缓存的值是查询结果对象的引用，而非深拷贝

### 缓存失效情况

1. 不同的 SqlSession 之间缓存数据互不影响
2. 同一 SqlSession 执行任何 INSERT、UPDATE、DELETE 操作会清空当前 SqlSession 的所有缓存
3. 同一 SqlSession 执行 flushCache=true 的查询会清空当前缓存
4. 同一 SqlSession 执行 commit 或 close 操作会清空缓存
5. 使用不同的查询参数或 SQL 语句时，无法命中缓存

### 源码实现

一级缓存的核心实现在 BaseExecutor 类中：

```java
@Override
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, 
                         ResultHandler resultHandler) throws SQLException {
    BoundSql boundSql = ms.getBoundSql(parameter);
    CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
    return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
}

@Override
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds,
                         ResultHandler resultHandler, CacheKey key, BoundSql boundSql) 
                         throws SQLException {
    // 省略部分代码...

    // 从一级缓存中查找
    list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
    if (list != null) {
        // 缓存命中，处理存储过程输出参数
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
    } else {
        // 缓存未命中，从数据库查询
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
    }

    // 省略部分代码...
    return list;
}
```

## 2. 二级缓存（Mapper 级别缓存）

二级缓存是 Mapper 级别的缓存，也称为命名空间缓存，作用范围更广。

### 工作原理

- 二级缓存默认关闭，需要手动配置开启
- 缓存作用域为同一个 Mapper（同一个命名空间），即不同 SqlSession 执行相同 Mapper 的相同查询会共享缓存
- 缓存存储在 MappedStatement 对象关联的 Cache 实例中
- 缓存的 key 组成与一级缓存类似
- 缓存的值是序列化后的对象拷贝，而非对象引用（除非配置为只读）

### 开启二级缓存

#### 1. 全局配置开启（mybatis-config.xml）

```xml
<settings>
    <setting name="cacheEnabled" value="true"/>
</settings>
```

#### 2. Mapper 级别开启（XxxMapper.xml）

```xml
<cache
    eviction="LRU"
    flushInterval="60000"
    size="512"
    readOnly="false"/>
```

#### 3. 查询语句开启缓存

```xml
<select id="getUser" resultType="User" useCache="true">
    SELECT * FROM user WHERE id = #{id}
</select>
```

### 二级缓存配置参数

| 参数 | 说明 | 默认值 |
|-----|------|--------|
| **eviction** | 缓存淘汰策略，可选 LRU、FIFO、SOFT、WEAK | LRU |
| **flushInterval** | 缓存刷新间隔，单位毫秒 | 不设置（不自动刷新） |
| **size** | 缓存大小 | 1024 个对象引用 |
| **readOnly** | 是否只读，true 时直接返回缓存对象引用（不安全但较快） | false |
| **blocking** | 是否阻塞，true 时会在高并发环境下保证只有一个线程到数据库查询 | false |

### 缓存失效情况

1. 执行任何 INSERT、UPDATE、DELETE 操作会清空关联命名空间的所有缓存
2. 执行 flushCache=true 的查询会清空关联命名空间的缓存
3. 缓存配置的 flushInterval 时间到期会自动刷新缓存
4. 缓存容量满时，会根据淘汰策略移除部分缓存

### 源码实现

二级缓存的核心实现在 CachingExecutor 装饰器中：

```java
@Override
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds,
                         ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
                         throws SQLException {
    Cache cache = ms.getCache();
    if (cache != null) {
        flushCacheIfRequired(ms);
        if (ms.isUseCache() && resultHandler == null) {
            ensureNoOutParams(ms, boundSql);
            @SuppressWarnings("unchecked")
            List<E> list = (List<E>) tcm.getObject(cache, key);
            if (list == null) {
                list = delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
                tcm.putObject(cache, key, list); // 存入二级缓存
            }
            return list;
        }
    }
    return delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
}
```

## 3. 一级缓存与二级缓存的区别

| 特性 | 一级缓存 | 二级缓存 |
|-----|---------|---------|
| **作用域** | SqlSession 级别 | Mapper(命名空间)级别 |
| **生命周期** | SqlSession 关闭时结束 | 应用级别，除非刷新或淘汰 |
| **共享范围** | 单个会话内 | 所有会话间共享（同一 Mapper） |
| **存储内容** | 对象引用 | 对象序列化拷贝(除非只读) |
| **默认状态** | 默认开启，无法关闭 | 默认关闭，需手动开启 |
| **并发安全性** | 无并发问题 | 需考虑并发和一致性 |
| **配置复杂度** | 无需配置 | 需要详细配置 |
| **适用场景** | 单线程应用 | 多线程、分布式环境 |

## 4. 自定义缓存

MyBatis 支持自定义缓存实现，可以集成第三方缓存产品：

### 实现 Cache 接口

```java
public class RedisCache implements Cache {
    private final String id;
    private RedisTemplate redisTemplate;

    public RedisCache(String id) {
        this.id = id;
        // 初始化 Redis 连接
    }

    @Override
    public String getId() {
        return this.id;
    }

    @Override
    public void putObject(Object key, Object value) {
        redisTemplate.opsForValue().set(key.toString(), value, 1, TimeUnit.HOURS);
    }

    @Override
    public Object getObject(Object key) {
        return redisTemplate.opsForValue().get(key.toString());
    }

    // 其他方法实现...
}
```

### 在 Mapper 中配置

```xml
<cache type="com.example.RedisCache"/>
```

## 5. 与 Spring 集成时的缓存

当 MyBatis 与 Spring 集成时，缓存行为有一些特殊情况：

- 默认情况下，Spring 管理的 SqlSession 是线程绑定的，每次操作都会关闭 SqlSession，导致一级缓存失效
- Spring 的事务管理可能影响缓存行为，同一事务内多次查询可能使用同一 SqlSession
- 可以考虑使用 Spring 自带的缓存机制(@Cacheable 等)代替 MyBatis 二级缓存

## 6. 缓存的最佳实践和注意事项

### 最佳实践

- 合理使用一级缓存，利用单次请求内的查询效率提升
- 谨慎使用二级缓存，特别是在复杂业务系统和分布式环境下
- 对于读多写少的数据，可以考虑开启二级缓存
- 对于实时性要求高的数据，应避免使用缓存或设置较短的过期时间
- 二级缓存实体类必须实现 Serializable 接口
- 考虑使用成熟的第三方缓存代替 MyBatis 二级缓存

### 常见问题和解决方案

#### 1. 缓存脏读问题

- **问题**：多个应用实例更新数据库，但缓存未同步更新
- **解决**：使用集中式缓存如 Redis，或实现缓存同步机制

#### 2. 缓存穿透问题

- **问题**：频繁查询不存在的数据，导致缓存无法发挥作用
- **解决**：对空结果也进行缓存，设置较短过期时间

#### 3. 缓存雪崩问题

- **问题**：大量缓存同时失效，导致数据库压力骤增
- **解决**：设置不同的过期时间，使用随机过期时间

#### 4. 一级缓存引起的幻读问题

- **问题**：同一 SqlSession 内，缓存数据与数据库不一致
- **解决**：关键操作前调用 clearCache()清除缓存

## 7. 面试准备建议

1. **深入理解缓存原理**：掌握一级缓存和二级缓存的工作机制
2. **源码阅读**：尝试阅读 MyBatis 缓存相关的源码，如 PerpetualCache、CachingExecutor 等
3. **实战案例准备**：准备实际项目中使用 MyBatis 缓存的案例，包括配置和效果
4. **问题排查经验**：总结缓存相关问题的排查和解决方法
5. **性能测试**：了解缓存对性能提升的实际效果
6. **集成案例**：准备 MyBatis 缓存与第三方缓存集成的案例

通过系统性的准备，你能够向面试官展示出对 MyBatis 缓存机制的深入理解和实践经验，增加面试通过的几率。

## 总结

MyBatis 的缓存机制是提升应用性能的重要手段：

- **一级缓存**适合单次请求内的重复查询优化，无需配置，自动生效
- **二级缓存**适合跨会话的数据共享，需要谨慎配置，注意并发和数据一致性问题
- **自定义缓存**可以集成 Redis 等第三方缓存，实现分布式缓存能力

在实际项目中，应根据业务特点和数据特性，合理选择缓存策略，并做好监控和异常处理，才能真正发挥缓存的性能优势。
