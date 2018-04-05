---
title: redis学习笔记(二) 性能保障
comments: true
date: 2018-01-16 22:23:42
categories:
- DB
tags:
- redis
---

本篇文章主要学习一下有哪些提高redis命令执行效率的方法，来最大限度的保障程序执行的性能，以前只是会用redis的一些基本的命令，但是对redis中的事务和流水线等的正确用法了解的不深。最近在工作中有用到，趁机好好学习一下。在学习之前，先看看几个基本的概念，包括redis事物，其中的命令包括multi, exec, watch,unwatch,和discard，流水线(pipline)等。



## Redis事务

#### 什么是Redis的基本事务

redis的基本事物需要用到multi和exec命令，这孩子那个事务可以让一个客户端在不被其他客户端打断的情况下执行多个命令。和关系数据库那种可以在执行的过程中进行回滚(rollback)的事务不同，在Redis里面，被multi和exec命令包围的所有命令会一个接一个地执行，知道所有命令都执行完毕为止。当一个事务执行完毕之后，Redis才会处理其他客户端的命令。

除了要使用multi和 exec命令之外，还要配合使用watch命令，有时候甚至还会用到unwatch和discard命令。在用户使用watch命令对键进行监视之后，直到用户执行exec命令的这段时间里面，如果有其他客户端抢先对被监视的键进行了替换，更新或删除操作，那么当用户尝试执行exec命令的时候，事务将失败，并返回一个错误。通过使用watch, multi/exec, unwatch/discard命令来确保自己在使用的数据没有发生变化来避免数据出错。

unwatch命令可以在watch命令之后，multi命令执行之前对连接进行重置(reset), 同样的，discard命令也可以在multi命令执行之后，exec命令执行之前对连接进行重置。这也就是说，用户在使用watch监视一个或多个键，接着使用multi命令开始一个新的事务，并将多个命令如队列到事务队列之后，仍然可以通过发送discard命令来取消watch命令并清空所有已入队命令。不过 discard很少使用。

**延迟执行事务提升性能**

因为redis在执行事务过程中，会延迟执行已入队的命令直到客户端发送exec命令为止。因此包括Python在内的客户端都会使用流水线来实现，一次性将所有事物包含的命令发送给Redis。

```python
def list_item(conn, itemid, sellerid, price):
    inventory = "inventory:%s" % sellerid
    item = "%s.%s" % (itemid, sellerid)
    end = time.time() + 5
    pipe = conn.pipeline()
    while time.time < end:
        try:
            pipe.watch(inventory) #监视用户包裹的变化
           
            if not pipe.sismeber(inventory, itemid):
                # 如果指定的闪频不在用户包裹里面,那么停止对包裹键的监视，
                # 并返回一个空值
                pipe.unwatch()
                return None
            pipe.multi()
            pipe.zadd("market:", item, price)
            pipe.srem(inventory, itemid)
            pipe.execute()
            # 如果执行execute方法没有引发WatchError异常，那么说明
            # 事务执行成功，并且对包裹键的监视也已经结束
            return True
        except redis.exception.WatchError:
            pass
    return False
```

### 流水线

redis中有一些命令可以接收多个参数，如MGET, MSET, HMSET, HMGET, RPUSH,和LPUSH, SADD, ZADD等。这些明空简化了哪些需要重复执行相同指令的操作，并且极大地提升了性能。初次之外，使用流水线，同样能获得极大的性能提升，并且可以同时执行多个不同的命令。

上面代码中，使用了multi和exec命令:

```python
pipe = conn.pipeline()
```

执行pipeline()方法传入True或者不传参数，那么客户端将使用Multi和exec包裹起用户的 所有命令。如果传入False为参数，那么客户端同样会收集所有要执行的命令。

```python
def update_token(conn, token, user, item=None):
    timestamp = time.time()
    # 设置流水线
    pipe = conn.pipeline(False)
    pipe.hset('login:', token, user)
    pipe.zadd('recent:', token, timestamp)
    if item:
        pipe.zadd('viewed:' + token, item, timestamp)
        pipe.zremrangebyrank('viewed:' + token, 0, timestamp)
        pipe.zincrby('viewed:', item, -1)
    # 执行那些被流水线包裹的命令
    pipe.execute()
```

