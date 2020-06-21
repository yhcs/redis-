
@[TOC](redis 基础数据结构之 hash表)
# 给新观众老爷的开场
大家好，我是弟弟！
最近读了一遍 黄健宏大佬的 **<<Redis 设计与实现>>**，对Redis 3.0版本有了一些认识
该书作者有一版添加了注释的 redis 3.0源码
[👉官方redis的github传送门](https://github.com/antirez/redis)。
[👉黄健宏大佬添加了注释的 redis 3.0源码传送门](https://github.com/huangz1990/redis-3.0-annotated.git)
网上说Redis代码写得很好，为了加深印象和学习redis大佬的代码写作艺术，了解工作中使用的redis 命令背后的源码逻辑，便有了写博客记录学习redis源码过程的想法。


# redis 的hash数据类型
redis 的hash数据类型，可以由 ziplist和 hash表 两种数据结构实现。
[上篇博客传送门:redis源码阅读 - 基础数据结构 之 ziplist](https://blog.csdn.net/a158372582/article/details/106107759)
下面来看下 hash表的数据结构👇
## 哈希表
redis里的hash表数据结构是使用链表地址法实现的。

### 字典结构
redis哈希表在内部封装在一个 dict对象里
```javascript
/*
 * 字典
 */
typedef struct dict {
    // 类型特定函数
    dictType *type;
    // 私有数据
    void *privdata;
    // 哈希表
    dictht ht[2];
    // rehash 索引
    // 当 rehash 不在进行时，值为 -1
    int rehashidx; /* rehashing not in progress if rehashidx == -1 */
    // 目前正在运行的安全迭代器的数量
    int iterators; /* number of iterators currently running */
} dict;
```

- dictType *type
>该字段存放着该字典哈希表上进行特定操作的具体函数
比如通过key计算hash值的hashFunction函数
```javascript
/*
 * 字典类型特定函数
 */
typedef struct dictType {
    // 计算哈希值的函数
    unsigned int (*hashFunction)(const void *key);
    // 复制键的函数
    void *(*keyDup)(void *privdata, const void *key);
    // 复制值的函数
    void *(*valDup)(void *privdata, const void *obj);
    // 对比键的函数
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);
    // 销毁键的函数
    void (*keyDestructor)(void *privdata, void *key);
    // 销毁值的函数
    void (*valDestructor)(void *privdata, void *obj);

} dictType;
```
- void *privdata
 > 配合 dictType *type 使用
  
- dictht ht[2]
>该dict对象有两个hash表
其中 key/value 一般都存在 dict->ht[0]里
当满足一定条件时，将对 ditc->ht[0]里的key/value 进行 rehash操作，此时 dict->ht[1] 将被用上。

 - int rehashidx
> rehash操作的标示字段，表示 ht[0] 上准备进行rehash操作的下标
 
- int iterators
 > 安全迭代器数量，如果>0，rehash操作将不会进行。
### 哈希表结构
- dictEntry **table
  指向哈希表数组
 - unsigned long size
   哈希表数组的长度
  - unsigned long sizemask
   哈希表掩码，用于计算key的hash值对应的哈希表数组下标
   - unsigned long used
	该哈希表中已有节点数量
```javascript
/* This is our hash table structure. Every dictionary has two of this as we
 * implement incremental rehashing, for the old to the new table. 
 * 哈希表
 * 每个字典都使用两个哈希表，从而实现渐进式 rehash 。
 */
typedef struct dictht {
    // 哈希表数组
    dictEntry **table;
    // 哈希表大小
    unsigned long size;
    // 哈希表大小掩码，用于计算索引值
    // 总是等于 size - 1
    unsigned long sizemask;
    // 该哈希表已有节点的数量
    unsigned long used;
} dictht;
```
### 哈希表数组中的节点
哈希表数组某一个下标存放的是 *dictEntry 链表的表头。
一个 dictEntry 里保存了一对 key/value、以及下一个dictEntry的指针
```javascript
typedef struct dictEntry {
    // 键
    void *key;
    // 值
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
    } v;
    // 指向下个哈希表节点，形成链表
    struct dictEntry *next;
} dictEntry;
```
# 从hset 命令与哈希表相关的部分
通过命令表找到 `hset` 命令的处理函数 hsetCommand

hsetCommand里 会优先尝试创建 ziplist数据结构实现的hash数据类型，因为ziplist省空间啊，
哈希表中的 field-value 被当成两个ziplist元素，一前一后加入到ziplist尾部

如果ziplist元素个数或者 单个元素的大小超过了ziplist的默认设置，将被转换为 哈希表实现。
转换过程为
1. 创建一个哈希表实现的 hash对象
2. 使用哈希对象迭代器遍历ziplist将所有key/value 取出后添加到哈希表中，
并将ziplist结构、与迭代器占用的空间释放

（可以在 t_hash.c/hashTypeConvertZiplist 里找到相应的源码逻辑）

## 创建哈希表方式实现的哈希对象
实际上就是创建了一个字典。
根据传入的dictType *type, void *privDataPtr 两个字段创建并初始化一个字典.
这里传入的dictType *type 是 &hashDictType， privDataPtr 为NULL
```javascript 
/* Hash type hash table (note that small hashes are represented with ziplists) */
dictType hashDictType = {
    dictEncObjHash,            /* hash function */
    NULL,                      /* key dup */
    NULL,                      /* val dup */
    dictEncObjKeyCompare,      /* key compare */
    dictRedisObjectDestructor, /* key destructor */
    dictRedisObjectDestructor  /* val destructor */
};
/* 将一个 ziplist 编码的哈希对象 o 转换成其他编码 */
void hashTypeConvertZiplist(robj *o, int enc) {
	...
	// 创建空白的新字典
    dict = dictCreate(&hashDictType, NULL);
    ...
}

/* Create a new hash table 
 * 创建一个新的字典
 */
dict *dictCreate(dictType *type,
        void *privDataPtr)
{
    dict *d = zmalloc(sizeof(*d));

    _dictInit(d,type,privDataPtr);

    return d;
}

 //* Initialize the hash table */
 //* 初始化哈希表
int _dictInit(dict *d, dictType *type,
        void *privDataPtr)
{
    // 初始化两个哈希表的各项属性值
    // 但暂时还不分配内存给哈希表数组
    _dictReset(&d->ht[0]);
    _dictReset(&d->ht[1]);
    // 设置类型特定函数
    d->type = type;
    // 设置私有数据
    d->privdata = privDataPtr;
    // 设置哈希表 rehash 状态
    d->rehashidx = -1;
    // 设置字典的安全迭代器数量
    d->iterators = 0;
    return DICT_OK;
}
static void _dictReset(dictht *ht)
{
    ht->table = NULL;
    ht->size = 0;
    ht->sizemask = 0;
    ht->used = 0;
}
```

## 在字典中加入 k/v 键值对
 hset 向hash对象加入 field/value 实际上就是向字典中加入 field/value 键值对
 redis 3.0逻辑如下
 1. 先尝试在字典中新增 field/value键值对
 2. 如果失败，则表示key存在，并更新value值
 

向字典中新增键值对的逻辑，源码写得很清晰
1. 先设置key，并返回一个 *dictEntry。
2. 如果 entry 为空，表示键已存在
3. 如果 key不存在，将值设置到entry上

```javascript
 * 尝试将给定键值对添加到字典中
 * 只有给定键 key 不存在于字典时，添加操作才会成功
 * 添加成功返回 DICT_OK ，失败返回 DICT_ERR
 * 最坏 T = O(N) ，平滩 O(1) 
int dictAdd(dict *d, void *key, void *val)
{
    // 尝试添加键到字典，并返回包含了这个键的新哈希节点
    // T = O(N)
    dictEntry *entry = dictAddRaw(d,key);

    // 键已存在，添加失败
    if (!entry) return DICT_ERR;

    // 键不存在，设置节点的值
    // T = O(1)
    dictSetVal(d, entry, val);

    // 添加成功
    return DICT_OK;
}
```
### 向字典中加入键
这里的逻辑就比较复杂了，先省略一些代码逻辑~，将目光聚焦在key的加入上。
这里的逻辑是
1. _dictKeyIndex函数 对key 计算哈希值，并判断哈希表中是否存在该key
2. 如果key已存在，则返回NULL
3. 判断字典是否在进行rehash操作，暂定为没有进行rehash操作，使用的是 d->ht[0]哈希表
4. 将创建的entry 设置到 ht->table[index]上。
5. 将key设置到entry上
```javascript
 * 尝试将键插入到字典中
 *
 * 如果键已经在字典存在，那么返回 NULL
 *
 * 如果键不存在，那么程序创建新的哈希节点，
 * 将节点和键关联，并插入到字典，然后返回节点本身。
 
dictEntry *dictAddRaw(dict *d, void *key)
{
    int index;
    dictEntry *entry;
    dictht *ht;
	...
    /* Get the index of the new element, or -1 if
     * the element already exists. */
    // 计算键在哈希表中的索引值
    // 如果值为 -1 ，那么表示键已经存在
    // T = O(N)
    if ((index = _dictKeyIndex(d, key)) == -1)
        return NULL;
    // T = O(1)
    /* Allocate the memory and store the new entry */
    // 如果字典正在 rehash ，那么将新键添加到 1 号哈希表
    // 否则，将新键添加到 0 号哈希表
    ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];
    // 为新节点分配空间
    entry = zmalloc(sizeof(*entry));
    // 将新节点插入到链表表头
    entry->next = ht->table[index];
    ht->table[index] = entry;
    // 更新哈希表已使用节点数量
    ht->used++;

    /* Set the hash entry fields. */
    // 设置新节点的键
    // T = O(1)
    dictSetKey(d, entry, key);

    return entry;
}
```
#### 键的hash值计算
计算键的hash值函数 就是 字典里 dict->type->hashFunction 指向的函数。
从创建哈希表实现的集合对象的代码，我们能找到这个 dict->type 就是变量 setDictType
对应的hash函数就是 dictEncObjHash ，而这个函数最终是使用了 MurmurHash2算法对key计算hash值

#### 查找键是否存在
因为一个字典里有两个哈希表
第一个哈希表是肯定要检查的，
第二个哈希表在 rehash操作开启后，需要检查。

而检查操作也很简单，通过hash值与哈希表掩码计算 下标idx,定位到链表头，
遍历链表，并调用dictCompareKeys，最终调用dictEncObjKeyCompare函数来比较两个 key是否相等。

```javascript
 * 返回可以将 key 插入到哈希表的索引位置
 * 如果 key 已经存在于哈希表，那么返回 -1
 *
 * 注意，如果字典正在进行 rehash ，那么总是返回 1 号哈希表的索引。
 * 因为在字典进行 rehash 时，新节点总是插入到 1 号哈希表。

static int _dictKeyIndex(dict *d, const void *key)
{
    unsigned int h, idx, table;
    dictEntry *he;

    /* Expand the hash table if needed */
    // 单步 rehash
    // T = O(N)
    if (_dictExpandIfNeeded(d) == DICT_ERR)
        return -1;

    /* Compute the key hash value */
    // 计算 key 的哈希值
    h = dictHashKey(d, key);
    // T = O(1)
    for (table = 0; table <= 1; table++) {

        // 计算索引值
        idx = h & d->ht[table].sizemask;

        /* Search if this slot does not already contain the given key */
        // 查找 key 是否存在
        // T = O(1)
        he = d->ht[table].table[idx];
        while(he) {
            if (dictCompareKeys(d, key, he->key))
                return -1;
            he = he->next;
        }

        // 如果运行到这里时，说明 0 号哈希表中所有节点都不包含 key
        // 如果这时 rehahs 正在进行，那么继续对 1 号哈希表进行 rehash
        if (!dictIsRehashing(d)) break;
    }

    // 返回索引值
    return idx;
}
```
#### 给entry设置键
因为哈希表方式实现的set集合，没有指定keyDup函数，将key直接赋值给了 entry->key
```javascript
// 设置给定字典节点的键
#define dictSetKey(d, entry, _key_) do { \
    if ((d)->type->keyDup) \
        entry->key = (d)->type->keyDup((d)->privdata, _key_); \
    else \
        entry->key = (_key_); \
} while(0)
```
### 给entry设置值
同样，因为哈希表方式实现的set集合，没有指定valDup函数，将value直接赋值给了 entry->v.val
```javascript
// 设置给定字典节点的值
#define dictSetVal(d, entry, _val_) do { \
    if ((d)->type->valDup) \
        entry->v.val = (d)->type->valDup((d)->privdata, _val_); \
    else \
        entry->v.val = (_val_); \
} while(0)
```
# 哈希表退化成链表怎么办？
在往字典中疯狂添加key/value键值对时
很有可能造成某个哈希数组元素上挂的链表过长从而影响查找性能。

redis在计算key的hash值之前的_dictExpandIfNeeded 函数里
对该哈希表进行适当的扩容操作
  👇
```javascript
int dictAdd(dict *d, void *key, void *val) {
    // 尝试添加键到字典，并返回包含了这个键的新哈希节点
    dictEntry *entry = dictAddRaw(d,key);
	...
}
dictEntry *dictAddRaw(dict *d, void *key) {
    ...
    if ((index = _dictKeyIndex(d, key)) == -1)
        return NULL;
	...
}
static int _dictKeyIndex(dict *d, const void *key) {
    ...
    if (_dictExpandIfNeeded(d) == DICT_ERR)
        return -1;
	
    /* Compute the key hash value */
    // 计算 key 的哈希值
    h = dictHashKey(d, key);
    ...
}
```
那我们来看下具体的扩容条件
在dict->rehashidx == -1 , 也就是字典没有正在进行扩容/缩容的前提下
 以下三种情况下对哈希表进行扩容并标记 dict->rehashidx 字段为0
 且扩展的哈希表的数组大小是第一个>= 所需要空间的 2的N次方
 方便使用哈希值与哈希表掩码进行 &操作计算哈希数组下标，(&操作取代了 %数组长度 取余)
 
  1. 字典已使用节点数和字典大小之间的比率接近 1：1，并且 dict_can_resize 为true
  2. 已使用节点数和字典大小之间的比率超过 dict_force_resize_ratio,该值默认为5
  3. 哈希表刚初始化完，是个空表，给哈希数组设置默认大小 DICT_HT_INITIAL_SIZE （4）

来看下源码👇
```javascript
* 查看字典是否正在 rehash
#define dictIsRehashing(ht) ((ht)->rehashidx != -1)

 * Expand the hash table if needed */
 * 根据需要，初始化字典（的哈希表），或者对字典（的现有哈希表）进行扩展
static int _dictExpandIfNeeded(dict *d)
{
    /* Incremental rehashing already in progress. Return. */
    // 渐进式 rehash 已经在进行了，直接返回
    if (dictIsRehashing(d)) return DICT_OK;

    /* If the hash table is empty expand it to the initial size. */
    // 如果字典（的 0 号哈希表）为空，那么创建并返回初始化大小的 0 号哈希表
    if (d->ht[0].size == 0) return dictExpand(d, DICT_HT_INITIAL_SIZE);

    /* If we reached the 1:1 ratio, and we are allowed to resize the hash
     * table (global setting) or we should avoid it but the ratio between
     * elements/buckets is over the "safe" threshold, we resize doubling
     * the number of buckets. */
    // 一下两个条件之一为真时，对字典进行扩展
    // 1）字典已使用节点数和字典大小之间的比率接近 1：1
    //    并且 dict_can_resize 为真
    // 2）已使用节点数和字典大小之间的比率超过 dict_force_resize_ratio
    if (d->ht[0].used >= d->ht[0].size &&
        (dict_can_resize ||
         d->ht[0].used/d->ht[0].size > dict_force_resize_ratio))
    {
        // 新哈希表的大小至少是目前已使用节点数的两倍
        // T = O(N)
        return dictExpand(d, d->ht[0].used*2);
    }

    return DICT_OK;
}
/* Expand or create the hash table */
/*
 * 创建一个新的哈希表，并根据字典的情况，选择以下其中一个动作来进行：
 *
 * 1) 如果字典的 0 号哈希表为空，那么将新哈希表设置为 0 号哈希表
 * 2) 如果字典的 0 号哈希表非空，那么将新哈希表设置为 1 号哈希表，
 *    并打开字典的 rehash 标识，使得程序可以开始对字典进行 rehash
 *
 * size 参数不够大，或者 rehash 已经在进行时，返回 DICT_ERR 。
 *
 * 成功创建 0 号哈希表，或者 1 号哈希表时，返回 DICT_OK 。
 *
 * T = O(N)
 */
int dictExpand(dict *d, unsigned long size)
{
    // 新哈希表
    dictht n; /* the new hash table */
    // 根据 size 参数，计算哈希表的大小
    // T = O(1)
    unsigned long realsize = _dictNextPower(size);
    /* the size is invalid if it is smaller than the number of
     * elements already inside the hash table */
    // 不能在字典正在 rehash 时进行
    // size 的值也不能小于 0 号哈希表的当前已使用节点
    if (dictIsRehashing(d) || d->ht[0].used > size)
        return DICT_ERR;
    /* Allocate the new hash table and initialize all pointers to NULL */
    // 为哈希表分配空间，并将所有指针指向 NULL
    n.size = realsize;
    n.sizemask = realsize-1;
    // T = O(N)
    n.table = zcalloc(realsize*sizeof(dictEntry*));
    n.used = 0;
    /* Is this the first initialization? If so it's not really a rehashing
     * we just set the first hash table so that it can accept keys. */
    // 如果 0 号哈希表为空，那么这是一次初始化：
    // 程序将新哈希表赋给 0 号哈希表的指针，然后字典就可以开始处理键值对了。
    if (d->ht[0].table == NULL) {
        d->ht[0] = n;
        return DICT_OK;
    }
    /* Prepare a second hash table for incremental rehashing */
    // 如果 0 号哈希表非空，那么这是一次 rehash ：
    // 程序将新哈希表设置为 1 号哈希表，
    // 并将字典的 rehash 标识打开，让程序可以开始对字典进行 rehash
    d->ht[1] = n;
    d->rehashidx = 0;
    return DICT_OK;
}
```
## rehash操作
从上述源码能看到，仅对哈希表进行了扩容操作相关操作，初始化一些字段 如 rehashidx，sizemask等
并没有进行任何rehash操作。

那 rehash操作跑哪儿去了呢？
redis对一个字典的rehash操作，不是一步到位的
不是一次性把该字典 dict->ht[0] 哈希表上所有哈希数组里的哈希数组元素全部重新哈希到 dict->ht[1]
而是将全部的rehash操作分散到对该字典操作的各个命令上了，每次进行"一步"哈希操作
(增加k/v，删除k/v，查找key，随机返回key)
```javascript
// 查看字典是否正在 rehash
#define dictIsRehashing(ht) ((ht)->rehashidx != -1)

dictEntry *dictAddRaw(dict *d, void *key, dictEntry **existing) {
    ...
    if (dictIsRehashing(d)) _dictRehashStep(d);
    ...
}

static void _dictRehashStep(dict *d) {
    if (d->iterators == 0) dictRehash(d,1);
}

```

如果该字典dict->rehashidx != -1,且无安全迭代器的情况下，单步哈希的流程大致如下👇
 1. 下标从dict->rehashidx开始，在  dict->ht[0].table 数组中找到第一个不为NULL的项
 2. 将 该项链表上的所有元素全部hash映射到 ditct->ht[1] 上
 3.  每重新映射一个元素， dict->ht[0].used --, dict->ht[1].used ++
 4. 该项链表处理完后，将 dict->rehashidx ++
 5. 达到rehash步数，结束循环
 

如果  dict->ht[0].used == 0，说明 dict->ht[0]中的元素已全部rehash到dict->ht[1],
     释放dict->ht[0].table 数组，
     设置 dict->ht[0] = dict->ht[1]
     重置 dict->ht[1]的字段
     设置 dict->rehashidx = -1
     rehash操作结束

3.0 源码逻辑如下👇
```javascript
/* Performs N steps of incremental rehashing. Returns 1 if there are still
 * keys to move from the old to the new hash table, otherwise 0 is returned.
 *
 * 执行 N 步渐进式 rehash 。
 *
 * 返回 1 表示仍有键需要从 0 号哈希表移动到 1 号哈希表，
 * 返回 0 则表示所有键都已经迁移完毕。
 *
 * Note that a rehashing step consists in moving a bucket (that may have more
 * than one key as we use chaining) from the old to the new hash table.
 *
 * 注意，每步 rehash 都是以一个哈希表索引（桶）作为单位的，
 * 一个桶里可能会有多个节点，
 * 被 rehash 的桶里的所有节点都会被移动到新哈希表。
 *
 * T = O(N)
 */
int dictRehash(dict *d, int n) {

    // 只可以在 rehash 进行中时执行
    if (!dictIsRehashing(d)) return 0;

    // 进行 N 步迁移
    // T = O(N)
    while(n--) {
        dictEntry *de, *nextde;

        /* Check if we already rehashed the whole table... */
        // 如果 0 号哈希表为空，那么表示 rehash 执行完毕
        // T = O(1)
        if (d->ht[0].used == 0) {
            // 释放 0 号哈希表
            zfree(d->ht[0].table);
            // 将原来的 1 号哈希表设置为新的 0 号哈希表
            d->ht[0] = d->ht[1];
            // 重置旧的 1 号哈希表
            _dictReset(&d->ht[1]);
            // 关闭 rehash 标识
            d->rehashidx = -1;
            // 返回 0 ，向调用者表示 rehash 已经完成
            return 0;
        }

        /* Note that rehashidx can't overflow as we are sure there are more
         * elements because ht[0].used != 0 */
        // 确保 rehashidx 没有越界
        assert(d->ht[0].size > (unsigned)d->rehashidx);

        // 略过数组中为空的索引，找到下一个非空索引
        while(d->ht[0].table[d->rehashidx] == NULL) d->rehashidx++;

        // 指向该索引的链表表头节点
        de = d->ht[0].table[d->rehashidx];
        /* Move all the keys in this bucket from the old to the new hash HT */
        // 将链表中的所有节点迁移到新哈希表
        // T = O(1)
        while(de) {
            unsigned int h;

            // 保存下个节点的指针
            nextde = de->next;

            /* Get the index in the new hash table */
            // 计算新哈希表的哈希值，以及节点插入的索引位置
            h = dictHashKey(d, de->key) & d->ht[1].sizemask;

            // 插入节点到新哈希表
            de->next = d->ht[1].table[h];
            d->ht[1].table[h] = de;

            // 更新计数器
            d->ht[0].used--;
            d->ht[1].used++;

            // 继续处理下个节点
            de = nextde;
        }
        // 将刚迁移完的哈希表索引的指针设为空
        d->ht[0].table[d->rehashidx] = NULL;
        // 更新 rehash 索引
        d->rehashidx++;
    }

    return 1;
}
```
# 字典的缩容
字典有扩容也有缩容
从字典中删除key后，若字典中的元素个数与字典大小满足一定关系，会触发缩容操作
缩绒条件是 
  * 哈希数组长度大于默认值DICT_HT_INITIAL_SIZE (4)，
且节点数量 与 字典哈希表数字大小的比例 小于10%

从 `hdel` 命令的处理函数 hdelCommand里能看到相关逻辑👇
```javascript

void hdelCommand(redisClient *c) {
    ...
    // 删除指定域值对
    for (j = 2; j < c->argc; j++) {
        if (hashTypeDelete(o,c->argv[j])) {
		   ...
        }
    }
	...
}

 /* 将给定 field 及其 value 从哈希表中删除
 * 删除成功返回 1 ，因为域不存在而造成的删除失败返回 0 。*/
int hashTypeDelete(robj *o, robj *field) {
    	...
    // 从字典中删除
     } else if (o->encoding == REDIS_ENCODING_HT) {
        if (dictDelete((dict*)o->ptr, field) == REDIS_OK) {
            deleted = 1;
            /* Always check if the dictionary needs a resize after a delete. */
            // 删除成功时，看字典是否需要收缩
            if (htNeedsResize(o->ptr)) dictResize(o->ptr);
        }
    } ...
}
// 返回给定字典的大小
#define dictSlots(d) ((d)->ht[0].size+(d)->ht[1].size)
// 返回字典的已有节点数量
#define dictSize(d) ((d)->ht[0].used+(d)->ht[1].used)
/* This is the initial size of every hash table */
/*
 * 哈希表的初始大小
 */
#define DICT_HT_INITIAL_SIZE     4
/* Hash table parameters */
#define REDIS_HT_MINFILL 10 /* Minimal hash table fill 10% */
int htNeedsResize(dict *dict)
{
    long long size, used;

    size = dictSlots(dict);
    used = dictSize(dict);
    return (size && used && size > DICT_HT_INITIAL_SIZE &&
            (used * 100 / size < REDIS_HT_MINFILL));
}

/* Resize the table to the minimal size that contains all the elements,
 * but with the invariant of a USED/BUCKETS ratio near to <= 1 
 * 缩小给定字典
 * 让它的已用节点数和字典大小之间的比率接近 1:1
 * 返回 DICT_ERR 表示字典已经在 rehash ，或者 dict_can_resize 为假。
 * 成功创建体积更小的 ht[1] ，可以开始 resize 时，返回 DICT_OK。 */
int dictResize(dict *d)
{
    int minimal;
    // 不能在关闭 rehash 或者正在 rehash 的时候调用
    if (!dict_can_resize || dictIsRehashing(d)) return DICT_ERR;
    // 计算让比率接近 1：1 所需要的最少节点数量
    minimal = d->ht[0].used;
    if (minimal < DICT_HT_INITIAL_SIZE)
        minimal = DICT_HT_INITIAL_SIZE;
    // 调整字典的大小
    // T = O(N)
    return dictExpand(d, minimal);
}
```



因为redis是单线程，这种将一整块儿rehash操作打散到各个命令中是为了避免单一命令执行时间过长，导致阻塞其他命令。
- 记得 server->db[x].dict 字典对象里记录的是该redis实例下第x号db里的所有k/v，
这个对象也是有扩容缩容的，如果这个全局k/v字典 扩容/缩容 一步到位，
但执行时间稍微长了一点，
将阻塞该 redis实例的第x号db上的，所有操作k/v的命令。
- server->db[x].dict的缩容 是通过时间事件触发 serverCron函数进行的
  每隔1ms会调用serverCron，其中会 判断 server->db[x].dict 字典是否需要缩容，
  如果字典需要缩容，每次进行"100步"缩容，如果耗时超过1ms，结束该字典本次的rehash操作

同样也因为redis是单线程，没有多线程带来的各种并发 读/写问题，
将rehash操作打散到各个命令里，逻辑上也算是比较清晰，
同时rehash操作没有执行完毕时，也能支持key的插入、删除、查找，且实现逻辑相对清晰。

# 字典的遍历，取hash对象里所有的k/v
对于稳定的字典，挨个哈希数组遍历就行了。

对于不稳定的字典(两次迭代器遍历期间，字典进行了扩容/缩容)
有点意思，准备另起一篇单独写一写

# 小结
1. 哈希数据类型具有存储一个对象的多个k/v的特性，
针对之前 使用字符串来存储对象(对象id做为key，整个对象序列化成字符串作为value)，
在获取对象的单个属性上，节省了对象的 序列化与反序列化 时间。

 # 往期博客回顾
1. [redis服务器的部分启动过程](https://blog.csdn.net/a158372582/article/details/106023071)
2.  [GET命令背后的源码逻辑](https://editor.csdn.net/md/?articleId=106035658)
3. [redis的基础数据结构之 sds](https://blog.csdn.net/a158372582/article/details/106063645)
4. [redis的基础数据结构之 list](https://blog.csdn.net/a158372582/article/details/106086284)
5. [redis的基础数据结构 之 ziplist](https://blog.csdn.net/a158372582/article/details/106107759)
