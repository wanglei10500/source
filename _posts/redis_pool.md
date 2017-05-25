---
title:  Redis  连接池
tags:
 - redis
categories: 问题解决
---

redis.clients.util.Pool.getResource会从JedisPool实例池返回一个可用redis连接。

Jedis实例管理来源于apache commons-pool GenericObjectPool

其中三个重要属性：

* MaxActive: 可用连接实例的最大数目,为负值时没有限制
* MaxIdle： 空闲连接实例的最大数目，为负值时没有限制。Idle的实例在使用前通常会通过org.apache.commons.pool.BasePoolableObjectFactory<T>的activateObject()方法使其变得可用。
* MaxWait: 等待可用连接的最大数目，单位毫秒

当连接池中没有可用连接(idle/active) 会等待maxWait时间。如过等待超时还没有可用连接，则抛出Could not get a resource from the pool
