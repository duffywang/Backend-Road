## CacheBuilder

为什么会出现本地缓存？

### 简介



##### 一句话介绍

Guava cache是一个支持高并发的线程安全的本地缓存，可自动加载数据进缓存，且具有多种缓存淘汰

##### 特点

- automatic loading of entries into the cache(自动将entry节点加载进缓存结构中)
- least-recently-used eviction when a maximum size is exceeded(当缓存的数据超过设置的最大值时，使用LRU算法移除)
- time-based expiration of entries, measured since last access or last write(具有基于entry节点上次被访问或更新的过期机制)
- keys automatically wrapped in [weak](http://docs.oracle.com/javase/7/docs/api/java/lang/ref/WeakReference.html?is-external=true) references(缓存Key封装在虚引用内)
- values automatically wrapped in [weak](http://docs.oracle.com/javase/7/docs/api/java/lang/ref/WeakReference.html?is-external=true) or [soft](http://docs.oracle.com/javase/7/docs/api/java/lang/ref/SoftReference.html?is-external=true) references(缓存value被封装在虚引用或软引用中)
- notification of evicted (or otherwise removed) entries(缓存被淘汰时通知机制)
- accumulation of cache access statistics(统计缓存使用过程中命中率、未命中率的统计)



##### 应用场景

​     愿意消耗一些内存空间来提升速度； 

​     能够预计某些key会被查询一次以上；

​     缓存中存放的数据总量不会超出内存容量。

### 使用例子

```java
   LoadingCache<Key, Graph> graphs = CacheBuilder.newBuilder()
       .maximumSize(10000)
       .expireAfterWrite(10, TimeUnit.MINUTES)
       .removalListener(MY_LISTENER)
       .build(
           new CacheLoader<Key, Graph>() {
             public Graph load(Key key) throws AnyException {
               return createExpensiveGraph(key);
             }
           });
```

### 方法介绍

##### newBuilder

构建一个默认配置的CacheBuilder

##### initalCapacity

初始内部哈希表最小容量大小，比如设为60，那么并发度就为8，即哈希表创建8个segments。在构建阶段评估一个合理的容量大小可以避免后续的扩容操作，但是设置一个很大值将浪费内存资源

##### concurrencyLevel

并发级别是指可以同时写缓存的线程数，设置过高了会浪费空间和时间，设置过低了会导致线程冲突

##### maximumSize

缓存中可包含entry的最大数量，当超过数量限制后将会淘汰entry，当缓存数量接近最大值，将淘汰那些不常使用的entry
当设置为0时，元素加载进缓存后就被淘汰，在需要暂时关闭缓存做数据测试时很方便

##### maximumWeight

设置缓存的最大权重
hat weight is only used to determine whether the cache is over capacity; it has no effect on selecting which entry should be evicted next
权重值只是用于衡量设置的值是否超过界限，对选择哪一个淘汰的entry没有影响

##### weigher

可以通过设置maximumSize值尝试回收最近没有使用或总体上很少使用的缓存项，除此之外，你还可以通过对缓存设定不同的权重weigher，来决定它的回收顺序决定回收

```java
LoadingCache<Key, PoiDto> poi = CacheBuilder.newBuilder()
        .maximumWeight(100000)
        .weigher(new Weigher<Key, PoiDto>() {
            public int weigh(Key k, PoiDto p) {
                return p.power();
            }
        })
```

##### weakKeys( ) 、weakValues( )、softValues( )

使用弱引用存储键。当键没有其它（强或软）引用时，缓存项可以被垃圾回收
使用弱引用存储值。当值没有其它（强或软）引用时，缓存项可以被垃圾回收
使用软引用存储值。软引用只有在响应内存需要时，才按照全局最近最少使用的顺序回收。


当使用以上三个方法后，将使用 == 比较缓存中元素（key/value）是否相等；已经被垃圾回收的entry还是会算进`Cache.size()`，但是无法读写操作

##### expireAfterWrite(long , TimeUnit  )

当缓存项在指定的时间段内没有写操作（创建或覆盖）就会被回收

##### expireAfterAccess(long , TimeUnit  )

当缓存项在指定的时间段内没有被读/写操作就会被回收，回收顺序按照
如`Cache.asMap.get(Object)` 、`Cache.asMap().put(K,V)`，但是`Cache.asMap()`不会更新访问时间


##### refreshAfterWrite(long , TimeUnit  )

缓存项上一次更新操作之后的多久会被刷新，通过内部`CacheLoader.reload(K, V)`实现
因为默认实现`CacheLoader.reload(K, V)`是 **同步**的 ，使用此方法的最好重写**异步**实现，否则刷新

##### removalListener(RemovalListener<? super K1,? super V1>  )

监听当缓存中的key被移除时触发的事件，可重写onRemoval 方法返回entry淘汰的原因，是容量限制、过期、被用户删除还是被用户替代

##### recordStats()

可以通过`Cache.stats()`查看命中的缓存数量、没有命中的缓存梳理、删除的缓存梳理





### 数据结构分析

插图Guava Cache

##### Segment

熟悉ConcurrentHashMap应该都知道Segment实现是依赖于ReentrantLock

```
 static class Segment<K, V> extends ReentrantLock
```

Segment内部数据如下

```java
//LocalCache
final LocalCache<K, V> map;
//segment存放元素的数量
volatile int count;
//修改、更新的数量，用来做弱一致性
int modCount;
//扩容用
int threshold;
//segment维护的数组，用来存放Entry。这里使用AtomicReferenceArray是因为要用CAS来保证原子性
volatile @MonotonicNonNull AtomicReferenceArray<ReferenceEntry<K, V>> table;
//如果key是弱引用的话，那么被GC回收后，就会放到ReferenceQueue，要根据这个queue做一些清理工作
final @Nullable ReferenceQueue<K> keyReferenceQueue;
//跟上同理
final @Nullable ReferenceQueue<V> valueReferenceQueue;
//如果一个元素新写入，则会记到这个队列的尾部，用来做expire
@GuardedBy("this")
final Queue<ReferenceEntry<K, V>> writeQueue;
//读、写都会放到这个队列，用来进行LRU替换算法
@GuardedBy("this")
final Queue<ReferenceEntry<K, V>> accessQueue;
//记录哪些entry被访问，用于accessQueue的更新。
final Queue<ReferenceEntry<K, V>> recencyQueue;
```



##### ReferenceEntry

```java
final K key;
final int hash;
//指向下一个Entry，说明这里用的链表（从上图可以看出）
final @Nullable ReferenceEntry<K, V> next;
//value
volatile ValueReference<K, V> valueReference = unset();
```




##### LocalCache

```java
 //Map的数组
 final Segment<K, V>[] segments;
 //并发量，即segments数组的大小
 final int concurrencyLevel;
 //key的比较策略，跟key的引用类型有关
 final Equivalence<Object> keyEquivalence;
 //value的比较策略，跟value的引用类型有关
 final Equivalence<Object> valueEquivalence;
 //key的强度，即引用类型的强弱
 final Strength keyStrength;
 //value的强度，即引用类型的强弱
 final Strength valueStrength;
 //访问后的过期时间
 final long expireAfterAccessNanos;
 //写入后的过期时间
 final long expireAfterWriteNanos;
 //刷新时间
 final long refreshNanos;
 //removal的事件队列，缓存过期后先放到该队列
 final Queue<RemovalNotification<K, V>> removalNotificationQueue;
 //设置的removalListener
 final RemovalListener<K, V> removalListener;
 //时间器
 final Ticker ticker;
 //创建Entry的工厂，根据引用类型不同
 final EntryFactory entryFactory;
```



### 源码分析

从LoadingCache的get方法开始

```java
    public V get(K key) throws ExecutionException {
      return localCache.getOrLoad(key);
    }
```

到LocalCache的`getOrLoad`


```java
V getOrLoad(K key) throws ExecutionException {
    return get(key, defaultLoader);
  }
V get(K key, CacheLoader<? super K, V> loader) throws ExecutionException {
    int hash = hash(checkNotNull(key));
    return segmentFor(hash).get(key, hash, loader);
  }

V get(K key, int hash, CacheLoader<? super K, V> loader) throws ExecutionException {
      checkNotNull(key);
      checkNotNull(loader);
      try {
        if (count != 0) { // read-volatile,缓存的个数
          // don't call getLiveEntry, which would ignore loading values
          ReferenceEntry<K, V> e = getEntry(key, hash);
          if (e != null) {
            long now = map.ticker.read();
            //获取没有过期的value,如果已经过期则返回null
            V value = getLiveValue(e, now);
            if (value != null) {
              //更新对应缓存最新读操作的时间及缓存命中率统计
              recordRead(e, now);
              statsCounter.recordHits(1);
              //判断是否刷新，如果需要刷新则异步刷新，返回旧值
              return scheduleRefresh(e, key, hash, value, now, loader);
            }
            ValueReference<K, V> valueReference = e.getValueReference();
            //如果entry过期了且value还在加载中，则等待直到加载完成。
            if (valueReference.isLoading()) {
              return waitForLoadingValue(e, key, valueReference);
            }
          }
        }

        // 执行到这说明entry要么为null要么已经过期，
        return lockedGetOrLoad(key, hash, loader);
      } catch (ExecutionException ee) {
        Throwable cause = ee.getCause();
        if (cause instanceof Error) {
          throw new ExecutionError((Error) cause);
        } else if (cause instanceof RuntimeException) {
          throw new UncheckedExecutionException(cause);
        }
        throw ee;
      } finally {
        postReadCleanup();
      }
    }


```





#####  

##### scheduleRefresh

```java
    V scheduleRefresh(ReferenceEntry<K, V> entry, K key, int hash, V oldValue, long now,
        CacheLoader<? super K, V> loader) {
      //1是否配置了refreshAfterWrite;2用writeTime判断是否达到刷新的时间;3是否在加载中，如果是则没必要再进行刷新,如果三者都满足则执行缓存刷新
      if (map.refreshes() && (now - entry.getWriteTime() > map.refreshNanos)
          && !entry.getValueReference().isLoading()) {
        V newValue = refresh(key, hash, loader, true);
        if (newValue != null) {
          return newValue;
        }
      }
      return oldValue;
    }
```

refresh 

```java
    V refresh(K key, int hash, CacheLoader<? super K, V> loader, boolean checkTime) {
      //为key插入一个LoadingValueReference，实质是把对应Entry的ValueReference替换为新建的LoadingValueReference
      final LoadingValueReference<K, V> loadingValueReference =
          insertLoadingValueReference(key, hash, checkTime);
      if (loadingValueReference == null) {
        return null;
      }
      //异步加载数据
      ListenableFuture<V> result = loadAsync(key, hash, loadingValueReference, loader);
      if (result.isDone()) {
        try {
          return Uninterruptibles.getUninterruptibly(result);
        } catch (Throwable t) {
          // don't let refresh exceptions propagate; error was already logged
        }
      }
      return null;
    }
```



insertLoadingValueReference返回新插入的值，如果value正在加载中则返回null

```java
    LoadingValueReference<K, V> insertLoadingValueReference(final K key, final int hash,
        boolean checkTime) {
      ReferenceEntry<K, V> e = null;
      lock();
      try {
        long now = map.ticker.read();
        preWriteCleanup(now);

        AtomicReferenceArray<ReferenceEntry<K, V>> table = this.table;
        int index = hash & (table.length() - 1);
        ReferenceEntry<K, V> first = table.get(index);

        // Look for an existing entry.
        for (e = first; e != null; e = e.getNext()) {
          K entryKey = e.getKey();
          if (e.getHash() == hash && entryKey != null
              && map.keyEquivalence.equivalent(key, entryKey)) {
            // We found an existing entry.

            ValueReference<K, V> valueReference = e.getValueReference();
            if (valueReference.isLoading()
                || (checkTime && (now - e.getWriteTime() < map.refreshNanos))) {
              // refresh is a no-op if loading is pending
              // if checkTime, we want to check *after* acquiring the lock if refresh still needs
              // to be scheduled
              return null;
            }

            // continue returning old value while loading
            ++modCount;
            LoadingValueReference<K, V> loadingValueReference =
                new LoadingValueReference<K, V>(valueReference);
            e.setValueReference(loadingValueReference);
            return loadingValueReference;
          }
        }

        ++modCount;
        LoadingValueReference<K, V> loadingValueReference = new LoadingValueReference<K, V>();
        e = newEntry(key, hash, first);
        e.setValueReference(loadingValueReference);
        table.set(index, e);
        return loadingValueReference;
      } finally {
        unlock();
        postWriteCleanup();
      }
    }
```





#####lockedGetOrLoad



##### 

### 实践


### 对比

##### HashMap 、ConcurrentHashMap

```java
public HashMap <String,String> cache = new HashMap<>();
public ConcurrentHashMap <String,String> cache = new ConcurrentHashMap<>();
```

首先是 JVM 缓存，也可以认为是堆缓存，其实就是创建一些全局变量，如 Map、List 之类的容器用于存放数据。

HashMap 比较适合像不需要淘汰机制、数据基本不变的场景，如利用反射，如果我们每次都通过反射去获取Method、field，性能肯定低，这时用HashMap缓存起获取到的数据性能可以提升不少

##### LRU-HashMap

针对上面两种方案中数据无法进行数据淘汰，内存无限制增长的情况，希望能将不常用的缓存进行删除，即出现具有淘汰策略的缓存，常见淘汰策略有FIFO、LRU、LFU ，最常见的最近最少使用算法(LRU，Least Recently Use)实现可依赖LinkedHashMap ，每次访问数据都会将数据放在队尾，如果达到阈值只需要淘汰队首的数据即可。



CacheBuilder  相比与HashMap和ConcurrentHashMap具有多种缓存失效方法，基于容量回收，设置最大的缓存容量；基于引用类型回收，可以选择弱引用和软引用；基于时间回收，可以选择访问时间和写入时间。



