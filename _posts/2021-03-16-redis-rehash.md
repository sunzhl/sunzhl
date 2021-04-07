---
title: 聊聊Redis键值存储结构以及Rehash机制
date: 2021-03-16 10:55:56
categories: Redis  
tags: 
  - Redis
  - NoSQL
---



### 一、键值对的结构

了解 `Redis` 朋友的都知道，`Redis` 是一种键值对 ( Key-Value Pair ) 数据库，在内存中键值对是以字典 ( Dict ) 的方式保存的，而字典的底层其实是通过 **哈希表** 来实现的。通过哈希表中的节点保存字典中的键值对。而这个哈希表的数据结构就是一个数组。

也就是说当我们添加或修改数据时，只需要计算出键的哈希值，然后，跟数组的大小取模，就可以很快的定位到它所对应的哈希桶的位置。所以，哈希表的最大好处就是我们可用**O(1)**的时间复杂度来快速查找键值对。(如下图所示)。

![image-20210316154858330](2021/03/16/redis-rehash/image-20210316154858330.png)

但是，这里存在一个问题，就是当存入的键之间计算的哈希值相同时，就会出现键冲突的问题。那么，Redis是如何解决的呢？

<!--more-->

### 二、键的哈希值冲突了怎么办

当往 `Redis`写入大量的数据时，不可避免的会出现键冲突的问题。这里的哈希冲突，也就是指，两个 key 的哈希值和哈希桶计算对应关系时，正好落在了同一个哈希桶中。毕竟，哈希桶的个数通常要少。

`Redis` 解决冲突的方式，跟 `Java`中的 `HashMap` 相同，也是以链表的方式解决的。也就是当多个键的哈希值相同，落到同一个哈希桶上时，它们之间以链表的形式保存。如下图所示：

![image-20210316160244468](2021/03/16/redis-rehash/image-20210316160244468.png)

但是，这里存在一个问题，哈希冲突链上的元素只能通过指针逐一查找再操作。如果哈希表里写入的数据越来越多，哈希冲突可能也会越来越多，这就会导致某些哈希冲突链过长，进而导致这个链上的元素查找耗时长，效率降低。对于追求“快”的 Redis 来说，这是不太能接受的。

所以，Redis 是通过扩容，也就是扩大哈希表的长度来解决此问题。

### 三、Redis中的Rehash

Redis 中是通过对哈希表进行 `Rehash` 操作，也就是增加现有哈希桶的数量，让主键增多的 数据能在更多的哈希桶之间分散保存，减少了每个桶中的元素数量，从而减少单个桶中的冲突。那它具体是怎么做的呢？

#### 1、字典结构

我们先通过Redis 源码看看 它的字典 ( dict )结构是如何实现。

```c
/* 源码文件 src\dict.h */

/* 哈希表
 * 每个字典都使用两个哈希表，以实现渐进式 rehash 。
 */
typedef struct dictht {
    dictEntry **table;       // 哈希表数组
    unsigned long size;      // 哈希表大小
    unsigned long sizemask;  // 哈希表大小掩码，用于计算索引值，总是等于 size - 1
    unsigned long used;      // 该哈希表 已有节点数量
} dictht;

// Redis 中的字典结构
typedef struct dict {
    dictType *type; // 类型特定函数
    void *privdata; // 私有数据
    dictht ht[2];  // 哈希表
    long rehashidx; /* Rehash 索引，当不在进行Rehash时 值为 -1 */
    unsigned long iterators; /* 目前正在运行的安全迭代器的数量 */
} dict;
```

由 Redis 源码我们不难发现，在 `dict` 结构中定义了两个 `dictht` ，也就是存在两个哈希表。Redis 这么做的目的就是为了使 rehash 操作更高效。Redis 正常情况下都是使用 哈希表1(  即 dict->ht[0] )，哈希表2( 即 dict->ht[1]  )并不会分配空间，只有当数据不断增多，需要进行 rehash 时采用用到 哈希表2。

#### 2、何时 rehash

在 Redis 中是跟  `Java` 中的 `HashMap` 一样，也是当哈希表中保存的元素达到某个阈值的时候，就会触发 rehash操作。同样我们也通过 Redis 源码来看看它是什么时候触发的。

```c
/*源码文件 src\dict.c */

// 添加元素
int dictAdd(dict *d, void *key, void *val){
     //  掉方法 dictAddRaw
    dictEntry *entry = dictAddRaw(d,key,NULL);
    // 省略部分代码
    return DICT_OK;
}

dictEntry *dictAddRaw(dict *d, void *key, dictEntry **existing){
    long index;
    dictEntry *entry;
    dictht *ht;

    if (dictIsRehashing(d)) _dictRehashStep(d); // 判断是否整在进行 rehash

    /* 
     * 获取新元素 在哈希表中的下标，如果已经存在返回 -1
     */
    if ((index = _dictKeyIndex(d, key, dictHashKey(d,key), existing)) == -1)
        return NULL;
    // 省略部分代码
    return entry;
}

static long _dictKeyIndex(dict *d, const void *key, uint64_t hash, dictEntry **existing){
    unsigned long idx, table;
    dictEntry *he;
    if (existing) *existing = NULL;

    /* Expand the hash table if needed */
    /* 判断 是否需要进行扩容 */
    if (_dictExpandIfNeeded(d) == DICT_ERR)
        return -1;
    
     // 省略部分代码
    return idx;
}

/*  判断是否需要进行扩容操作 */
static int _dictExpandIfNeeded(dict *d){
    /*  如果正在进行 rehash 则直接返回 OK. */
    if (dictIsRehashing(d)) return DICT_OK;

    /*  如果当前 哈希表1 为空，即第一次添加元素，则直接创建 */
    if (d->ht[0].size == 0) return dictExpand(d, DICT_HT_INITIAL_SIZE);

    /*
     * 如果当前已有元素大于等于哈希表的大小，并且允许 rehash 或者已经
     * 超过了安全阈值，则进行扩容操作
     * 
     */
    if (d->ht[0].used >= d->ht[0].size &&
        (dict_can_resize ||
         d->ht[0].used/d->ht[0].size > dict_force_resize_ratio))
    {
        return dictExpand(d, d->ht[0].used*2);
    }
    return DICT_OK;
}
```

> 说明：
>
> 1、d->ht[0].used >= d->ht[0].size  判断哈希表1 中的已有元素是否大于等于哈希表的长度。
>
> 2、dict_can_resize 是否允许进行 rehash 的标准 ，0-不允许，1-允许。当 redis 正在进行 生成 RDB 快照或者 重新AOF 文件时，此值设置为0
>
> 3、d->ht[0].used/d->ht[0].size > dict_force_resize_ratio 判断哈希表 中的已有元素与哈希表长度的比率是超过安全比率 dict_force_resize_ratio。 dict_force_resize_ratio 值为 5。

总结，当 Redis 中哈希表中的已有元素个数大于等于哈希表的长度，并且 Redis 不在处于 正在生产 RDB快照或者重写AOF文件，或者 哈希表已有元素个与哈希表的长度的比率大于 5 时，就会触发 rehash 操作。

> <span style="color: #FF0000">**强调：Redis中的阈值时固定，不可改的。**</span>

#### 3、rehash 步骤

扩展和收缩哈希表的工作可以通过执行rehash（重新散列）操作来完成，Redis 中执行 rehash 的步骤如下：

1. 给哈希表2 ( ht[1] )分配的空间。这个哈希表的空间大小取决于要执行的操作，以及ht[0]当前包含的键值对数量（也即是ht[0].used属性的值）：
   - 如果执行的是扩展操作，那么 ht[1] 的大小为第一个大于等于 ht[0].used*2 的 2 n（2的n次方幂）；
   - 如果执行的是收缩操作，那么 ht[1] 的大小为第一个大于等于 ht[0].used 的2 n

2. 将保存在 ht[0] 中的所有键值对 rehash 到 ht[1] 上面：rehash指的是重新计算键的哈希值和索引值，然后将键值对放置到ht[1]哈希表的指定位置上。

3. 当 ht[0] 包含的所有键值对都迁移到了 ht[1] 之后（ht[0]变为空表），释放ht[0]，将ht[1]设置为ht[0]，并在ht[1]新创建一个空白哈希表，为下一次rehash做准备。

上面的步骤看似很简单，但是，我们知道 Redis 是单线程的，那么如果一次性把哈希表1中的数据全部迁移完，势必会造成 Redis 线程阻塞，无法服务其他请求，此时，Redis 就无法快速访问数据了。

为了避免这个问题，Redis 采用了 **渐进式rehash**

#### 4、渐进式 rehash

何为渐进式rehash ？简单来说就是在第二步拷贝数据时，Redis 不是集中式的一次性完成的，而是分多次、渐进式地完成的。 Redis 是把数据的迁移任务分散到了每次的请求中。

- **操作步骤**

1. 在字典中维护了一个 rehashidx 变量，当开始 rehash 时，他的值设置为 0，也就是表示从哈希表1，索引0 开始。
2. 在每次的 增、删、改、查操作时，除了执行指定操作之外，还会判断是否正在 rehash，即 rehashidx != -1，如果是，就会顺带把 rehashidx 索引位置的 元素迁移到 哈希表2中。
3. 随着操作不断的执行，最终会在某个时间点上，哈希表1 中的所有键值都会被 rehash 到哈希表2中。这时会把rehashidx的值设置为 -1，表示 rehash 操作已完成。

如下图所示：

![image-20210317110125739](2021/03/16/redis-rehash/image-20210317110125739.png)

- **源码分析**

  由上面介绍的 rehash 触发条件的源码中我们可知当满足条件后回调用 `dictExpand` 方法，那么我们看下此方法的详细实现

  ```c
  int dictExpand(dict *d, unsigned long size){
      /* the size is invalid if it is smaller than the number of
       * elements already inside the hash table */
      if (dictIsRehashing(d) || d->ht[0].used > size)
          return DICT_ERR;
  
      dictht n; /* the new hash table */
      // 计算新表格的长度
      unsigned long realsize = _dictNextPower(size); 
  
      /* Rehashing to the same table size is not useful. */
      if (realsize == d->ht[0].size) return DICT_ERR;
  
      /* Allocate the new hash table and initialize all pointers to NULL */
      /* 分配一个新的 哈希表， 并且初始化所有的指针指向 null */
      n.size = realsize;
      n.sizemask = realsize-1;
      n.table = zcalloc(realsize*sizeof(dictEntry*));
      n.used = 0;
  
      /* Is this the first initialization? If so it's not really a rehashing
       * we just set the first hash table so that it can accept keys. */
      if (d->ht[0].table == NULL) { // 第一次创建
          d->ht[0] = n;
          return DICT_OK;
      }
  
      /* Prepare a second hash table for incremental rehashing */
      d->ht[1] = n;
      d->rehashidx = 0; // 把 rehash 标志设置为 0，
      return DICT_OK;
  }
  ```

  > 此方法主要实现了以下功能：
  >
  > 1. 计算需要扩容的新的哈希表的大小
  > 2. 给哈希表分配空间，并且所有的指针都指向 null
  > 3. 把rehash索引 rehashidx 设置为 0，也就是说 rehash 是从哈希表 1 的 0 位桶开始拷贝。

  接下来我们在看看，Redis中 增删改查的代码中是如何进行 rehash 的。

  ```c
  /* 增加方法 */
  dictEntry *dictAddRaw(dict *d, void *key, dictEntry **existing){
      long index;
      dictEntry *entry;
      dictht *ht;
      // 判断是否整在进行 rehash
      if (dictIsRehashing(d)) _dictRehashStep(d); 
      
      // 省略部分代码
  }
   /* 查询方法 */
  dictEntry *dictFind(dict *d, const void *key){
      dictEntry *he;
      uint64_t h, idx, table;
  
      if (d->ht[0].used + d->ht[1].used == 0) return NULL; /* dict is empty */
       // 判断是否整在进行 rehash
      if (dictIsRehashing(d)) _dictRehashStep(d);
  
      // 省略部分代码   
  }
  
  /* 删除方法 */
  static dictEntry *dictGenericDelete(dict *d, const void *key, int nofree) {
      uint64_t h, idx;
      dictEntry *he, *prevHe;
      int table;
  
      if (d->ht[0].used == 0 && d->ht[1].used == 0) return NULL;
     
      // 判断是否整在进行 rehash
      if (dictIsRehashing(d)) _dictRehashStep(d);
      
      // 省略部分代码
  }
  ```

  有增、删、查的方法我们可以发现所有的方法中都要 `if (dictIsRehashing(d)) _dictRehashStep(d);`  这段代码，此行代码就是 判断当前是否正在进行 rehash 也就是 `rehashidx != -1 ` 则进行一步 rehash 操作。

#### 5、定时辅助 rehash 

由于，Redis 是渐进的方式进行 rehash 的，所以，在 Redis 中的键值对很多的情况下，那么，哈希表1 和 哈希表2 将共存很长时间，Redis 在渐进式 rehash 的同时，还会在 Redis 空闲时也会定时辅助进行 rehash 。

下面我们通过 Redis 源码，看看是如何实现的。

- 调度方法 `serverCron`  

```c
int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData) {
    
    // 省略部分代码
    
      /* We need to do a few operations on clients asynchronously. */
    clientsCron();

    /* 处理 Redis 数据库后端操作，再次方法中执行定时 rehash 操作 */
    databasesCron();

    // 省略部分代码
}
```

- databaseCron 方法

  ```c
  void databasesCron(void) {
      /* 处理过期键 */
      if (server.active_expire_enabled) {
          if (server.masterhost == NULL) {
              activeExpireCycle(ACTIVE_EXPIRE_CYCLE_SLOW);
          } else {
              expireSlaveKeys();
          }
      }
  
      /* Defrag keys gradually. */
      if (server.active_defrag_enabled)
          activeDefragCycle();
  
      /* 如果当前系统没有在 生成 RDB 快照或 重写 AOF 文件，则进 */
      if (server.rdb_child_pid == -1 && server.aof_child_pid == -1) {
          
          // 省略部分代码
  
          /* Rehash 操作  */
          if (server.activerehashing) {
              for (j = 0; j < dbs_per_call; j++) {
                  int work_done = incrementallyRehash(rehash_db);
                  if (work_done) {
                      /* If the function did some work, stop here, we'll do
                       * more at the next cron loop. */
                      break;
                  } else {
                      /* If this db didn't need rehash, we'll try the next one. */
                      rehash_db++;
                      rehash_db %= server.dbnum;
                  }
              }
          }
      }
  }
  ```
  
  > 说明：
  >
  > 可以在 redis.conf 配置文件中配置 activerehashing 的值是否启动，定时辅助 rehash，默认为 yes 启动。

### 四、扩展知识

#### 1、rehash 期间如何查找值

因为在进行渐进式 rehash 的过程中，字典会同时使用 ht[0] 和 ht[1] 两个哈希表，所以在渐进式 rehash 进行期间，字典的删除（delete）、查找（find）、更新（update）等操作会在两个哈希表上进行。首先会现在 ht[0] 上查找，如果不存在，则到 ht[1] 上查找。

#### 2、rehash期间有新增怎么存

在渐进式 rehash 执行期间，新添加到字典的键值对一律会被保存到 ht[1] 里面，而 ht[0] 则不再进行任何添加操作，这一措施保证了 ht[0] 包含的键值对数量会只减不增，并随着 rehash 操作的执行而最终变成空表。

#### 3、生成 RDB 文件或 重写 AOF 期间为什么不能 rehash 操作

这是因为在 Redis 中是通过创建子进程的方式，生成 RDB 快照或 重写 AOF 文件的，而大多数操作系统都采用写时复制（copy-on-write）技术来优化子进程的使用效率。因为 rehash 操作需要父进程申请新的哈希表，所有在 生成RDB或 AOF重写 期间，会造成父进程大量的 COW操作，影响父进程的性能。

> 说明：
>
>    如果负载因子大于一定值(固定为 5) 时，即时正在进行 生成RDB 或 重写 AOF 也会进行 rehash 操作。

#### 4、为什么rehash期间，允许RDB和AOF rewrite？

因为 rehash 已经开始，说明新的哈希表内存已经申请完成了，之后的 rehash 过程只是迁移哈希桶下的数据指针，不涉及到内存申请了，所以 RDB 和 AOF rewrite 没影响。

### 五、小结

在 Redis 中是以 哈希表的方式来保存键值对数据的，但是随着键值对的增多，会出现 哈希冲突的情况，这种情况，Redis 是以链表的方式解决哈希冲突的。当链表变得很长时，会影响 Redis 的查找性能，为了减小 链表的长度，Redis 采用了 rehash 操作，也就是把扩大当前哈希表的长度，Redis 在 rehash 是不是一次性rehash ，而是采用了渐进式方式，这样可以解决长时间阻塞，在 渐进式 rehash 的同时，Redis 在空闲时间也会进行 1Ms 的定时 rehash。