---
layout: post
title:  "Redis Mass Insertion & python example"
categories: redis python
tags: redis python
author: 大可
---


## 为什么需要单独提出大量插入的问题
- 一个一个地将key/value插入Redis是不合适的，在大批量插入的场景下，大量时间被浪费在网络往返上，效率低下
- Redis是单线程应用，一个特别大的原子请求将阻塞整个应用，使得在其他client看来，Redis暂时不可用
- Redis虽然采取了IO多路复用的优化手段，但是仅有少数client能够获益，不是解决问题的根本手段

## Redis批提交的几种方式

### mset
直接支持写入多条的redis命令

- 优势
    - 性能优异
- 劣势
    - 存在部分失败的情况，需要根据返回值进一步处理
    - 在单次插入大量数据的情况下，效率差
    - mset无法设置过期时间，需要另行操作

### pipeline
pipeline是一种允许一次提交多条命令的方式，且可以控制每次提交至服务端的包的大小

- 优势
    - 一次性提交多个不同的命令
    - 分批发送至服务端，设置合适的包大小，不会长时间阻塞服务端
- 劣势
    - 由于会分多次发送，那么在不同批次之间别的client可能会执行命令，使得不具备事务保证（事实上存在transaction参数）

### transaction
事务会将多条命令一次性提交给服务端，且保证原子性，不会在执行中途插入其他client的操作

- 优势
    - 保证原子性
    - 乐观锁机制
- 劣势
    - 性能略低于pipeline
    - 采用乐观锁的话效率低下

## pipeline python示例

```python
import redis


conn = redis.Redis(
    host=redis_host,
    port=int(redis_port),
    db=db_name,
    socket_timeout=DEFAULT_SOCKET_TIMEOUT
)


def pipeline_mset(conn, mapping: dict, expired_seconds=None, batch_size=100):
    """
    通过pipeline方式设置多个key/value
    :param mapping: 要缓存的键值对
    :param expired_seconds: 过期时间
    :param batch_size: 单批提交数量
    :return:
    """
    pipe = conn.pipeline(transaction=False)
    if expired_seconds is None:
        expired_seconds = 60

    count = 0
    for key, value in mapping.items():
        value = pickle.dumps(value)

        pipe.set(name=key, value=value, ex=expired_seconds)

        count += 1
        if count == batch_size:
            pipe.execute()
            count = 0

    if count != 0:
        pipe.execute()
```

## 参考
- [Redis Mass Insertion](https://redis.io/topics/mass-insert)
- [pipeline速度提升建议](https://stackoverflow.com/questions/32149626/how-to-insert-billion-of-data-to-redis-efficiently)
- [Redis批量操作分析](https://www.jianshu.com/p/75137d23ae4a)

