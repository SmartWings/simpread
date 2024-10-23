> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.cnblogs.com](https://www.cnblogs.com/booksea/p/17155464.html)

本文已收录至 Github，推荐阅读 👉 [Java 随想录](https://github.com/ZhengShuHai/JavaRecord)

微信公众号：[Java 随想录](https://mmbiz.qpic.cn/mmbiz_jpg/jC8rtGdWScMuzzTENRgicfnr91C5Bg9QNgMZrxFGlGXnTlXIGAKfKAibKRGJ2QrWoVBXhxpibTQxptf8MsPTyHvSg/0?wx_fmt=jpeg)

这篇文章来介绍一个本地缓存框架：**Caffeine Cache**。被称为现代缓存之王。Spring Boot 1.x 版本中的默认本地缓存是 Guava Cache。在 Spring5 (SpringBoot 2.x) 后，Spring 官方放弃了 Guava Cache 作为缓存机制，而是使用性能更优秀的 Caffeine 作为默认缓存组件，这对于 Caffeine 来说是一个很大的肯定。

目录

*   [Caffeine Cache 介绍](#caffeine-cache介绍)
    *   [淘汰算法](#淘汰算法)
*   [SpringBoot 集成 Caffeine Cache](#springboot集成caffeine-cache)
    *   [Cache 类型](#cache类型)
        *   [Cache](#cache)
        *   [Loading Cache](#loading-cache)
        *   [Async Cache](#async-cache)
        *   [Async Loading Cache](#async-loading-cache)
    *   [驱逐策略](#驱逐策略)
        *   [基于大小的过期方式](#基于大小的过期方式)
        *   [基于时间的过期方式](#基于时间的过期方式)
        *   [基于引用的过期方式](#基于引用的过期方式)
    *   [写入外部存储](#写入外部存储)
    *   [被动刷新机制](#被动刷新机制)
    *   [统计](#统计)
    *   [SpringBoot 中集成 Caffeine](#springboot中集成caffeine)
        *   [配置文件的方式注入相关参数](#配置文件的方式注入相关参数)
        *   [Bean 的方式注入相关参数](#bean的方式注入相关参数)
        *   [注解使用姿势](#注解使用姿势)
        *   [缓存同步模式](#缓存同步模式)

Caffeine Cache 介绍
-----------------

Caffeine 是基于 JAVA 8 的高性能缓存库， 因使用 **Window TinyLfu** 回收策略，提供了一个近乎最佳的命中率。

### 淘汰算法

**FIFO(First In First Out)：先进先出**

优先淘汰掉最先缓存的数据、是最简单的淘汰算法。缺点是如果先缓存的数据使用频率比较高的话，那么该数据就不停地进出，导致它的缓存命中率比较低。

**LRU(Least Recently Used)：最近最久未使用**

优先淘汰掉最久未访问到的数据。缺点是不能很好地应对突发流量。比如一个数据在一分钟内的前 59 秒访问很多次，而在最后 1 秒没有访问，但是有一批冷门数据在最后一秒进入缓存，那么热点数据就会被淘汰掉。

**LFU(Least Frequently Used)：最近最少频率使用**

先淘汰掉最不经常使用的数据，需要维护一个表示使用频率的字段。如果访问频率比较高的话，频率字段会占据一定的空间。并且如果数据访问模式随时间有变，LFU 的频率信息无法随之变化，因此早先频繁访问的记录可能会占据缓存，而后期访问较多的记录则无法被命中。

**W-TinyLFU 算法**

Caffeine 使用了 W-TinyLFU 算法，看名字就能大概猜出来，它是 LFU 的变种，解决了 LRU 和 LFU 上述的缺点 。

W-TinyLFU 具体是如何实现的，怎么解决 LRU 和 LFU 的缺点的，有兴趣可以网上搜索相关文章。这里不发散开来细讲，本篇文章重点是介绍具体使用。

SpringBoot 集成 Caffeine Cache
----------------------------

首先导入依赖

```
        <!-- https://mvnrepository.com/artifact/com.github.ben-manes.caffeine/caffeine -->
        <dependency>
            <groupId>com.github.ben-manes.caffeine</groupId>
            <artifactId>caffeine</artifactId>
            <version>3.1.1</version>
        </dependency>


```

Caffeine 类使用了建造者模式，有如下配置参数：

*   **expireAfterWrite**：写入间隔多久淘汰；
*   **expireAfterAccess**：最后访问后间隔多久淘汰；
*   **refreshAfterWrite**：写入后间隔多久刷新，支持异步刷新和同步刷新，如果和 expireAfterWrite 组合使用，能够保证即使该缓存访问不到、也能在固定时间间隔后被淘汰，否则如果单独使用容易造成 OOM，**使用 refreshAfterWrite 时必须指定一个 CacheLoader**；
*   **expireAfter**：自定义淘汰策略，该策略下 Caffeine 通过时间轮算法来实现不同 key 的不同过期时间；
*   **maximumSize**：缓存 key 的最大个数；
*   **weakKeys**：key 设置为弱引用，在 GC 时可以直接淘汰；
*   **weakValues**：value 设置为弱引用，在 GC 时可以直接淘汰；
*   **softValues**：value 设置为软引用，在内存溢出前可以直接淘汰；
*   **executor**：选择自定义的线程池，默认的线程池实现是 ForkJoinPool.commonPool()；
*   **maximumWeight**：设置缓存最大权重；
*   **weigher**：设置具体 key 权重；
*   **recordStats**：缓存的统计数据，比如命中率等；
*   **removalListener**：缓存淘汰监听器；

### Cache 类型

Caffeine 共提供了四种类型的 Cache，对应着四种加载策略。

#### Cache

最普通的一种缓存，无需指定加载方式，需要手动调用 put() 进行加载。需要注意的是，put() 方法对于已存在的 key 将进行覆盖。

在获取缓存值时，如果想要在缓存值不存在时，原子地将值写入缓存，则可以调用 get(key, k -> value) 方法，该方法将避免写入竞争。

调用 invalidate() 方法，将手动移除缓存。

多线程情况下，当使用 get(key, k -> value) 时，如果有另一个线程同时调用本方法进行竞争，则后一线程会被阻塞，直到前一线程更新缓存完成；而若另一线程调用 getIfPresent() 方法，则会立即返回 null，不会被阻塞。

```
Cache<String, String> cache = Caffeine.newBuilder().build();

cache.getIfPresent("1");    // null
cache.get("1", k -> 1);    // 1
cache.getIfPresent("1");    //1
cache.set("1", "2");
cache.getIfPresent("1");    //2


```

#### Loading Cache

LoadingCache 是一种自动加载的缓存。其和普通缓存不同的地方在于，**当缓存不存在 / 缓存已过期时，若调用 get() 方法，则会自动调用 CacheLoader.load() 方法加载最新值**。调用 getAll() 方法将遍历所有的 key 调用 get()，除非实现了 CacheLoader.loadAll() 方法。

**使用 LoadingCache 时，需要指定 CacheLoader，并实现其中的 load() 方法供缓存缺失时自动加载**。

多线程情况下，当两个线程同时调用 get()，则后一线程将被阻塞，直至前一线程更新缓存完成。

```
LoadingCache<String, Object> cache = Caffeine.newBuilder().build(new CacheLoader<String, Object>() {
            @Override
            public @Nullable Object load(@NonNull String s) {
                return "从数据库读取";
            }

            @Override
            public @NonNull Map<@NonNull String, @NonNull Object> loadAll(@NonNull Iterable<? extends @NonNull String> keys) {
                return null;
            }
        });

cache.getIfPresent("1"); // null
cache.get("1");          // 从数据库读取
cache.getAll(keys);        // null


```

**LoadingCache 特别实用，我们可以在 load 方法里配置逻辑，缓存不存在的时候去我们的数据库加载，可以实现多级缓存**。

#### Async Cache

AsyncCache 是 Cache 的一个变体，其响应结果均为 CompletableFuture，通过这种方式，AsyncCache 对异步编程模式进行了适配。默认情况下，缓存计算使用 ForkJoinPool.commonPool() 作为线程池，如果想要指定线程池，则可以覆盖并实现 Caffeine.executor(Executor) 方法。

多线程情况下，当两个线程同时调用 get(key, k -> value)，则会返回**同一个 CompletableFuture** 对象。由于返回结果本身不进行阻塞，可以根据业务设计自行选择阻塞等待或者非阻塞。

```
AsyncCache<String, String> cache = Caffeine.newBuilder().buildAsync();

CompletableFuture<String> completableFuture = cache.get(key, k -> "1");
completableFuture.get(); // 阻塞，直至缓存更新完成


```

#### Async Loading Cache

显然这是 Loading Cache 和 Async Cache 的功能组合。AsyncLoadingCache 支持以异步的方式，对缓存进行自动加载。

类似 LoadingCache，同样需要指定 CacheLoader，并实现其中的 load() 方法供缓存缺失时自动加载，该方法将自动在 ForkJoinPool.commonPool() 线程池中提交。如果想要指定 Executor，则可以实现 AsyncCacheLoader().asyncLoad() 方法。

```
AsyncLoadingCache<String, String> cache = Caffeine.newBuilder()
        .buildAsync(new AsyncCacheLoader<String, String>() {
            @Override
            // 自定义线程池加载
            public @NonNull CompletableFuture<String> asyncLoad(@NonNull String key, @NonNull Executor executor) {
                return null;
            }
        })
    //或者使用默认线程池加载（和上面方式二者选其一）
        .buildAsync(new CacheLoader<String, String>() {
            @Override  
            public String load(@NonNull String key) throws Exception {
                return "456";
            }
        });

CompletableFuture<String> completableFuture = cache.get(key); // CompletableFuture<String>
completableFuture.get(); // 阻塞，直至缓存更新完成


```

**Async Loading Cache 也特别实用，有些业务场景我们 Load 数据的时间会比较长，这时候就可以使用 Async Loading Cache，避免 Load 数据阻塞**。

### 驱逐策略

Caffeine 提供了 3 种回收策略：基于大小回收，基于时间回收，基于引用回收

#### 基于大小的过期方式

基于大小的回收策略有两种方式：一种是基于缓存大小，一种是基于权重。

```
// 根据缓存的计数进行驱逐
LoadingCache<String, Object> cache = Caffeine.newBuilder()
    .maximumSize(10000)
    .build(new CacheLoader<String, Object>() {

            @Override
            public @Nullable Object load(@NonNull String s) {
                return "从数据库读取";
            }

            @Override
            public @NonNull Map<@NonNull String, @NonNull Object> loadAll(@NonNull Iterable<? extends @NonNull String> keys) {
                return null;
            }
        });


// 根据缓存的权重来进行驱逐，权重低的会被优先驱逐
LoadingCache<String, Object> cache1 = Caffeine.newBuilder()
    .maximumWeight(10000)
    .weigher(new Weigher<String, Object>() {
                    @Override
                    public @NonNegative int weigh(@NonNull String s, @NonNull Object o) {
                        return 0;
                    }
                })
    .build(new CacheLoader<String, Object>() {
            @Override
            public @Nullable Object load(@NonNull String s) {
                return "从数据库读取";
            }

            @Override
            public @NonNull Map<@NonNull String, @NonNull Object> loadAll(@NonNull Iterable<? extends @NonNull String> keys) {
                return null;
            }
        });


```

**maximumWeight 与 maximumSize 不可以同时使用**。

#### 基于时间的过期方式

```
// 基于不同的到期策略进行退出
LoadingCache<String, Object> cache = Caffeine.newBuilder()
    .expireAfter(new Expiry<String, Object>() {
        @Override
        public long expireAfterCreate(String key, Object value, long currentTime) {
            return TimeUnit.SECONDS.toNanos(seconds);
        }

        @Override
        public long expireAfterUpdate(@Nonnull String s, @Nonnull Object o, long l, long l1) {
            return 0;
        }

        @Override
        public long expireAfterRead(@Nonnull String s, @Nonnull Object o, long l, long l1) {
            return 0;
        }
    }).build(key -> function(key));


```

Caffeine 提供了三种定时驱逐策略：

expireAfterAccess(long, TimeUnit)：在最后一次访问或者写入后开始计时，在指定的时间后过期。假如一直有请求访问该 key，那么这个缓存将一直不会过期。

expireAfterWrite(long, TimeUnit)：在最后一次写入缓存后开始计时，在指定的时间后过期。

expireAfter(Expiry)：自定义策略，过期时间由 Expiry 实现独自计算。

#### 基于引用的过期方式

Java 中四种引用类型

<table><thead><tr><th>引用类型</th><th>被垃圾回收时间</th><th>用途</th><th>生存时间</th></tr></thead><tbody><tr><td>强引用 Strong Reference</td><td>从来不会</td><td>对象的一般状态</td><td>JVM 停止运行时终止</td></tr><tr><td>软引用 Soft Reference</td><td>在内存不足时</td><td>对象缓存</td><td>内存不足时终止</td></tr><tr><td>弱引用 Weak Reference</td><td>在垃圾回收时</td><td>对象缓存</td><td>gc 运行后终止</td></tr><tr><td>虚引用 Phantom Reference</td><td>从来不会</td><td>可以用虚引用来跟踪对象被垃圾回收器回收的活动，当一个虚引用关联的对象被垃圾收集器回收之前会收到一条系统通知</td><td>JVM 停止运行时终止</td></tr></tbody></table>

```
// 当key和value都没有引用时驱逐缓存
LoadingCache<String, Object> cache = Caffeine.newBuilder()
    .weakKeys()
    .weakValues()
    .build(key -> function(key));

// 当垃圾收集器需要释放内存时驱逐
LoadingCache<String, Object> cache1 = Caffeine.newBuilder()
    .softValues()
    .build(key -> function(key));


```

**注意：AsyncLoadingCache 不支持弱引用和软引用。**

Caffeine.weakKeys()： 使用弱引用存储 key。如果没有其他地方对该 key 有强引用，那么该缓存就会被垃圾回收器回收。

Caffeine.weakValues() ：使用弱引用存储 value。如果没有其他地方对该 value 有强引用，那么该缓存就会被垃圾回收器回收。

Caffeine.softValues() ：使用软引用存储 value。当内存满了过后，软引用的对象以将使用最近最少使用 (least-recently-used) 的方式进行垃圾回收。由于使用软引用是需要等到内存满了才进行回收，所以我们通常建议给缓存配置一个使用内存的最大值。

**Caffeine.weakValues() 和 Caffeine.softValues() 不可以一起使用。**

### 写入外部存储

CacheWriter 方法可以将缓存中所有的数据写入到第三方，避免数据在服务重启的时候丢失。

```
LoadingCache<String, Object> cache = Caffeine.newBuilder()
    .writer(new CacheWriter<String, Object>() {
        @Override public void write(String key, Object value) {
            // 写入到外部存储
        }
        @Override public void delete(String key, Object value, RemovalCause cause) {
            // 删除外部存储
        }
    })
    .build(key -> function(key));


```

### 被动刷新机制

试想这样一种情况：当缓存运行过程中，有些缓存值我们需要定期进行刷新变更。

使用刷新机制 **refreshAfterWrite()**，刷新机制只支持 **LoadingCache 和 AsyncLoadingCache**。

refreshAfterWrite() 方法是一种被动更新，它必须设置 CacheLoad，key 过期后并不立即刷新 value：

1、当过期后第一次调用 get() 方法时，得到的仍然是过期值，同时异步地对缓存值进行刷新；

2、当过期后第二次调用 get() 方法时，才会得到更新后的值。

通过覆写 CacheLoader.reload() 方法，将在刷新时使得旧缓存值参与其中。

```
Caffeine.newBuilder()
                .refreshAfterWrite(1,TimeUnit.MINUTES)
                .build(new CacheLoader<String, Object>() {
                    @Override
                    public @Nullable Object load(@NonNull String s) throws Exception {
                        return null;
                    }

                    @Override
                    public @Nullable Object reload(@NonNull String key, @NonNull Object oldValue) throws Exception {
                        return null;
                    }
                });


```

### 统计

Caffeine 内置了数据收集功能，通过 Caffeine.recordStats() 方法，可以打开数据收集。这样 Cache.stats() 方法将会返回当前缓存的一些统计指标，例如：

*   hitRate()：查询缓存的命中率
*   evictionCount()：被驱逐的缓存数量
*   averageLoadPenalty()：新值被载入的平均耗时

```
Cache<String, String> cache = Caffeine.newBuilder().recordStats().build();
cache.stats(); // 获取统计指标


```

### SpringBoot 中集成 Caffeine

我们可以直接在 SpringBoot 中通过创建 Bean 的方法进行应用。除此以外，SpringBoot 还自带了一些更加方便的缓存配置与管理功能。

除了上述的 Caffeine 依赖，我们还需导入：

```
<!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-cache -->
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-cache</artifactId>
</dependency>


```

**记得在启动类上添加 @EnableCaching 注解**

#### 配置文件的方式注入相关参数

```
spring:
  cache:
    type: caffeine
    cache-names:
    - userCache
    caffeine:
      spec: maximumSize=1024,refreshAfterWrite=60s


```

#### Bean 的方式注入相关参数

```
@Configuration
public class CacheConfig {


    /**
     * 创建基于Caffeine的Cache Manager
     * 初始化一些key存入
     * @return
     */
    @Bean
    @Primary
    public CacheManager caffeineCacheManager() {
        SimpleCacheManager cacheManager = new SimpleCacheManager();
        ArrayList<CaffeineCache> caches = Lists.newArrayList();
        List<CacheBean> list = setCacheBean();
        for(CacheBean cacheBean : list){
            caches.add(new CaffeineCache(cacheBean.getKey(),
                    Caffeine.newBuilder().recordStats()
                            .expireAfterWrite(cacheBean.getTtl(), TimeUnit.SECONDS)
                            .maximumSize(cacheBean.getMaximumSize())
                            .build()));
        }
        cacheManager.setCaches(caches);
        return cacheManager;
    }


    /**
     * 初始化一些缓存的 key
     * @return
     */
    private List<CacheBean> setCacheBean(){
        List<CacheBean> list = Lists.newArrayList();
        CacheBean userCache = new CacheBean();
        userCache.setKey("userCache");
        userCache.setTtl(60);
        userCache.setMaximumSize(10000);

        CacheBean deptCache = new CacheBean();
        deptCache.setKey("deptCache");
        deptCache.setTtl(60);
        deptCache.setMaximumSize(10000);

        list.add(userCache);
        list.add(deptCache);

        return list;
    }

    class CacheBean {
        private String key;
        private long ttl;
        private long maximumSize;

        public String getKey() {
            return key;
        }

        public void setKey(String key) {
            this.key = key;
        }

        public long getTtl() {
            return ttl;
        }

        public void setTtl(long ttl) {
            this.ttl = ttl;
        }

        public long getMaximumSize() {
            return maximumSize;
        }

        public void setMaximumSize(long maximumSize) {
            this.maximumSize = maximumSize;
        }
    }

}


```

使用这种方式，可以同时在缓存管理器中添加多个缓存。需要注意的是，**SimpleCacheManager 只能使用 Cache 和 LoadingCache，异步缓存将无法支持**。

#### 注解使用姿势

还可以通过注解的方式进行使用，相关的常用注解包括：

*   **@Cacheable**：表示该方法支持缓存。当调用被注解的方法时，如果对应的键已经存在缓存，则不再执行方法体，而从缓存中直接返回。当方法返回 null 时，将不进行缓存操作。
*   **@CachePut**：表示执行该方法后，其值将作为最新结果更新到缓存中。**每次都会执行该方法**。
*   **@CacheEvict**：表示执行该方法后，将触发缓存清除操作。
*   **@Caching**：用于组合前三个注解，例如：

```
@Caching(cacheable = @Cacheable("users"),
         evict = {@CacheEvict("cache2"), @CacheEvict(value = "cache3", allEntries = true)})
public User find(Integer id) {
    return null;
}


```

这类注解也同时可以标记在一个类上，表示该类的所有方法都支持对应的缓存注解。

@Cacheable 常用的注解属性如下：

*   **cacheNames/value**：缓存组件的名字，即 cacheManager 中缓存的名称。
*   **key**：缓存数据时使用的 key。默认使用方法参数值，也可以使用 SpEL 表达式进行编写。
*   **keyGenerator**：和 key 二选一使用。
*   **cacheManager**：指定使用的缓存管理器。
*   **condition**：在方法执行开始前检查，在符合 condition 的情况下，进行缓存。
*   **unless**：在方法执行完成后检查，在符合 unless 的情况下，不进行缓存。
*   **sync**：是否使用同步模式。若使用同步模式，在多个线程同时对一个 key 进行 load 时，其他线程将被阻塞。

Spring Cache 提供了一些供我们使用的 SpEL 上下文数据，下表直接摘自 Spring 官方文档：

<table><thead><tr><th>名称</th><th>位置</th><th>描述</th><th>示例</th></tr></thead><tbody><tr><td>methodName</td><td>root 对象</td><td>当前被调用的方法名</td><td><code>#root.methodname</code></td></tr><tr><td>method</td><td>root 对象</td><td>当前被调用的方法</td><td><code>#root.method.name</code></td></tr><tr><td>target</td><td>root 对象</td><td>当前被调用的目标对象实例</td><td><code>#root.target</code></td></tr><tr><td>targetClass</td><td>root 对象</td><td>当前被调用的目标对象的类</td><td><code>#root.targetClass</code></td></tr><tr><td>args</td><td>root 对象</td><td>当前被调用的方法的参数列表</td><td><code>#root.args[0]</code></td></tr><tr><td>caches</td><td>root 对象</td><td>当前方法调用使用的缓存列表</td><td><code>#root.caches[0].name</code></td></tr><tr><td>Argument Name</td><td>执行上下文</td><td>当前被调用的方法的参数，如 findArtisan(Artisan artisan), 可以通过 #artsian.id 获得参数</td><td><code>#artsian.id</code></td></tr><tr><td>result</td><td>执行上下文</td><td>方法执行后的返回值（仅当方法执行后的判断有效，如 unless cacheEvict 的 beforeInvocation=false）</td><td><code>#result</code></td></tr></tbody></table>

#### 缓存同步模式

@Cacheable 注解支持配置同步模式。在不同的 Caffeine 配置下，对是否开启同步模式进行观察。

<table><thead><tr><th>Caffeine 缓存类型</th><th>是否开启同步</th><th>多线程读取不存在 / 已驱逐的 key</th><th>多线程读取待刷新的 key</th></tr></thead><tbody><tr><td>Cache</td><td>否</td><td>各自独立执行被注解方法</td><td>-</td></tr><tr><td>Cache</td><td>是</td><td>线程 1 执行被注解方法，线程 2 被阻塞，直至缓存更新完成</td><td>-</td></tr><tr><td>LoadingCache</td><td>否</td><td>线程 1 执行<code>load()</code>，线程 2 被阻塞，直至缓存更新完成</td><td>线程 1 使用老值立即返回，并异步更新缓存值；线程 2 立即返回，不进行更新。</td></tr><tr><td>LoadingCache</td><td>是</td><td>线程 1 执行被注解方法，线程 2 被阻塞，直至缓存更新完成</td><td>线程 1 使用老值立即返回，并异步更新缓存值；线程 2 立即返回，不进行更新。</td></tr></tbody></table>

从上面的总结可以看到，sync 开启或关闭，在 Cache 和 LoadingCache 中的表现是不一致的：

*   Cache 中，sync 表示是否需要所有线程同步等待
*   LoadingCache 中，sync 表示在读取不存在 / 已驱逐的 key 时，是否执行被注解方法

本篇文章就到这里，感谢阅读，如果本篇博客有任何错误和建议，欢迎给我留言指正。文章持续更新，可以关注公众号第一时间阅读。

![](https://mmbiz.qpic.cn/mmbiz_jpg/jC8rtGdWScMuzzTENRgicfnr91C5Bg9QNgMZrxFGlGXnTlXIGAKfKAibKRGJ2QrWoVBXhxpibTQxptf8MsPTyHvSg/0?wx_fmt=jpeg)