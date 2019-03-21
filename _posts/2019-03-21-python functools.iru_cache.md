---
layout: post
title:  "python functools.iru_cache"
categories: python
tags: python python-method std-module
author: 大可
---

### LRU算法：最近最少使用算法
- 对于该算法，对于每一个缓存值设置一个年龄位，当hit缓存记录时，更新击中记录为最新，当缓存项达到maxsize时，用该次miss项替代年龄位最老的缓存项，一般的实现方式（https://en.wikipedia.org/wiki/Cache_replacement_policies#Least_recently_used_(LRU)）

- JAVA中使用LinkedHashMap实现key value存储，使用链表顺序来表明年龄

### 用法
```
from functools import lru_cache

@lru_cache(maxsize=32, typed=False)
def test_lru_cache(info1, info2):
    return info1 *info2 * 2

if __name__ == "__main__":
    # typed参数默认为False,在该情况下，相等的值会被认为是触发同一个缓存值，而当typed参数为True时，不同类型的值即便是
    # 相等的，也会被分别缓存，占用两个entry
    test_lru_cache(1, 2)
    print(test_lru_cache.cache_info()) #CacheInfo(hits=0, misses=1, maxsize=32, currsize=1)
    test_lru_cache(1.0, 2.0)
    print(test_lru_cache.cache_info()) #CacheInfo(hits=1, misses=1, maxsize=32, currsize=1)

    # 对于使用位置参数和关键字参数两种方式来调用函数，会被分别缓存至不同的entry中
    test_lru_cache(info1=1, info2=2)
    print(test_lru_cache.cache_info()) #CacheInfo(hits=1, misses=2, maxsize=32, currsize=2)
    test_lru_cache(1, info2=2)
    print(test_lru_cache.cache_info()) #CacheInfo(hits=1, misses=3, maxsize=32, currsize=3)

    # 对于多参数的被修饰函数，参数顺序的改变也会使得被缓存到不同的entry中
    test_lru_cache(info2=2, info1=1)
    print(test_lru_cache.cache_info()) #CacheInfo(hits=1, misses=4, maxsize=32, currsize=4)

    # 综上所述，对于函数结果的缓存与参数传入模式有关
```

### 源码阅读
```
_CacheInfo = namedtuple("CacheInfo", ["hits", "misses", "maxsize", "currsize"])

class _HashedSeq(list):
    """ This class guarantees that hash() will be called no more than once
        per element.  This is important because the lru_cache() will hash
        the key multiple times on a cache miss.

    """

    __slots__ = 'hashvalue'

    def __init__(self, tup, hash=hash):
        self[:] = tup
        self.hashvalue = hash(tup)

    def __hash__(self):
        return self.hashvalue

def _make_key(args, kwds, typed,
             kwd_mark = (object(),),
             fasttypes = {int, str, frozenset, type(None)},
             tuple=tuple, type=type, len=len):
    """Make a cache key from optionally typed positional and keyword arguments

    The key is constructed in a way that is flat as possible rather than
    as a nested structure that would take more memory.

    If there is only a single argument and its data type is known to cache
    its hash value, then that argument is returned without a wrapper.  This
    saves space and improves lookup speed.

    """
    # All of code below relies on kwds preserving the order input by the user.
    # Formerly, we sorted() the kwds before looping.  The new way is *much*
    # faster; however, it means that f(x=1, y=2) will now be treated as a
    # distinct call from f(y=2, x=1) which will be cached separately.
    key = args
    
    # 将位置参数元组和关键字参数转为的元组结合，即此时key是包含位置参数和关键字参数的元组，两种参数标识形式不同
    # 故可以解释为什么不同的参数传入模式会分别缓存到不同的单元中
    if kwds:
        key += kwd_mark
        for item in kwds.items():
            key += item
    # 当typed为真时，将位置参数及关键字参数的值的类型的元组与之前key元组结合，此时key中包含三种内容，但是结构是扁平
    # 的，从而节约内存
    if typed:
        key += tuple(type(v) for v in args)
        if kwds:
            key += tuple(type(v) for v in kwds.values())
            
    # 当函数仅有一个参数且忽略类型，同时参数类型可知，即为int, str, frozenset, type(None)中的某种时，
    # 直接返回，来节约空间和提高检索速度
    elif len(key) == 1 and type(key[0]) in fasttypes:
        return key[0]
    return _HashedSeq(key)

# 三层参数装饰器
def lru_cache(maxsize=128, typed=False):
    """Least-recently-used cache decorator.

    If *maxsize* is set to None, the LRU features are disabled and the cache
    can grow without bound.

    If *typed* is True, arguments of different types will be cached separately.
    For example, f(3.0) and f(3) will be treated as distinct calls with
    distinct results.

    Arguments to the cached function must be hashable.

    View the cache statistics named tuple (hits, misses, maxsize, currsize)
    with f.cache_info().  Clear the cache and statistics with f.cache_clear().
    Access the underlying function with f.__wrapped__.

    See:  http://en.wikipedia.org/wiki/Cache_algorithms#Least_Recently_Used

    """

    # Users should only access the lru_cache through its public API:
    #       cache_info, cache_clear, and f.__wrapped__
    # The internals of the lru_cache are encapsulated for thread safety and
    # to allow the implementation to change (including a possible C version).

    # Early detection of an erroneous call to @lru_cache without any arguments
    # resulting in the inner function being passed to maxsize instead of an
    # integer or None.
    if maxsize is not None and not isinstance(maxsize, int):
        raise TypeError('Expected maxsize to be an integer or None')

    def decorating_function(user_function):
        wrapper = _lru_cache_wrapper(user_function, maxsize, typed, _CacheInfo)
        return update_wrapper(wrapper, user_function)

    return decorating_function

def _lru_cache_wrapper(user_function, maxsize, typed, _CacheInfo):
    # Constants shared by all lru cache instances:
    
    # 闭包的自由变量
    sentinel = object()          # unique object used to signal cache misses
    make_key = _make_key         # build a key from the function arguments
    
    # 链表中元素为长度为四的列表，分别为：指向前一个元素、指向后一个元素、根据传入参数生成的key、所缓存的函数返回值
    PREV, NEXT, KEY, RESULT = 0, 1, 2, 3   # names for the link fields

    # 使用字典来缓存，字典中所缓存的value为链表中的一个单元，即四元列表
    cache = {}
    hits = misses = 0
    full = False
    cache_get = cache.get    # bound method to lookup a key or return None
    cache_len = cache.__len__  # get cache size without calling len()
    lock = RLock()           # because linkedlist updates aren't threadsafe
    
    # 构建双向循环链表来描述LRU算法中的年龄，初始状态为仅有root元素，且之后始终保持root为标识，其NEXT元素为
    # 最老单元，即最有可能接下来被替代的单元，其PREV为最新单元
    root = []                # root of the circular doubly linked list
    root[:] = [root, root, None, None]     # initialize by pointing to self

    # 当maxsize为0时，不做缓存，仅将misses数递增
    if maxsize == 0:

        def wrapper(*args, **kwds):
            # No caching -- just a statistics update after a successful call
            nonlocal misses
            result = user_function(*args, **kwds)
            misses += 1
            return result

    # 当maxsize为None时，意即缓存所有调用过的结果，此时无需区分新老关系，故双向循环链表无用，直接缓存
    elif maxsize is None:

        def wrapper(*args, **kwds):
            # Simple caching without ordering or size limit
            nonlocal hits, misses
            key = make_key(args, kwds, typed)
            result = cache_get(key, sentinel)
            if result is not sentinel:
                hits += 1
                return result
            result = user_function(*args, **kwds)
            cache[key] = result
            misses += 1
            return result
    
    else:

        def wrapper(*args, **kwds):
            # Size limited caching that tracks accesses by recency
            nonlocal root, hits, misses, full
            key = make_key(args, kwds, typed)
            with lock:
                link = cache_get(key)
                
                # 当hit缓存时，先将该单元从链表中拿出，即将前一个单元与后一个单元相连
                # 然后将该单元置于最新单元处，即root的PREV
                
                if link is not None:
                    # Move the link to the front of the circular queue
                    link_prev, link_next, _key, result = link
                    link_prev[NEXT] = link_next
                    link_next[PREV] = link_prev
                    last = root[PREV]
                    last[NEXT] = root[PREV] = link
                    link[PREV] = last
                    link[NEXT] = root
                    hits += 1
                    return result
            result = user_function(*args, **kwds)
            
            # 此处可以释放锁的原因在于对链表及字典的一次完整操作已经完成
            with lock:
                # 由于在之前释放了锁且判断过目标不在缓存中，那么当此判断为真时，说明在释放锁之后
                # 该目标被缓存，由于已经错过hit的逻辑，此处做简单处理，认为缓存miss
                # 这意味着在多线程操作的情形下，可能出现同样的两次访问，均miss的情况
                if key in cache:
                    # Getting here means that this same key was added to the
                    # cache while the lock was released.  Since the link
                    # update is already done, we need only return the
                    # computed result and update the count of misses.
                    pass
                    
                    
                # 如果miss，且链表长度达到maxsize，则将root的key、value赋值为要缓存的key、value，然后将
                # root[NEXT]即最老缓存的key、value清空，并将其对象引用赋给root，即保证root的含义，最后去除缓存
                elif full:
                    # Use the old root to store the new key and result.
                    oldroot = root
                    oldroot[KEY] = key
                    oldroot[RESULT] = result
                    # Empty the oldest link and make it the new root.
                    # Keep a reference to the old key and old result to
                    # prevent their ref counts from going to zero during the
                    # update. That will prevent potentially arbitrary object
                    # clean-up code (i.e. __del__) from running while we're
                    # still adjusting the links.

                    # root始终为空，等待填充,故当循环链表满时，链表有maxsize+1个元素

                    root = oldroot[NEXT]
                    oldkey = root[KEY]
                    oldresult = root[RESULT]
                    root[KEY] = root[RESULT] = None
                    # Now update the cache dictionary.
                    del cache[oldkey]
                    # Save the potentially reentrant cache[key] assignment
                    # for last, after the root and links have been put in
                    # a consistent state.
                    cache[key] = oldroot
                    
                # 如果miss但是未达maxsize，则新建要缓存的四元列表，置于root和root[PREV]之间，
                # 即意味着仅当full满足时才会挪动root的位置
                    
                else:
                    # Put result in a new link at the front of the queue.
                    last = root[PREV]
                    link = [last, root, key, result]
                    last[NEXT] = root[PREV] = cache[key] = link
                    # Use the cache_len bound method instead of the len() function
                    # which could potentially be wrapped in an lru_cache itself.
                    full = (cache_len() >= maxsize)
                misses += 1
            return result

    def cache_info():
        """Report cache statistics"""
        with lock:
            return _CacheInfo(hits, misses, maxsize, cache_len())

    def cache_clear():
        """Clear the cache and cache statistics"""
        nonlocal hits, misses, full
        with lock:
            cache.clear()
            root[:] = [root, root, None, None]
            hits = misses = 0
            full = False

    wrapper.cache_info = cache_info
    wrapper.cache_clear = cache_clear
    return wrapper
```

{:toc}