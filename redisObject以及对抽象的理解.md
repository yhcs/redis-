
@[TOC](redisObject 以及对抽象的理解)
# 给新观众老爷的开场
大家好，我是弟弟！
最近读了一遍 黄健宏大佬的 **<<Redis 设计与实现>>**，对Redis 3.0版本有了一些认识
该书作者有一版添加了注释的 redis 3.0源码
[👉官方redis的github传送门](https://github.com/antirez/redis)。
[👉黄健宏大佬添加了注释的 redis 3.0源码传送门](https://github.com/huangz1990/redis-3.0-annotated.git)


网上说Redis代码写得很好，为了加深印象和学习redis大佬的代码写作艺术，了解工作中使用的redis 命令背后的源码逻辑，便有了写博客记录学习redis源码过程的想法。


# redisObject
看过前几期的观众老爷，都会常常见到一个叫 robj 的结构体
比如在k/v空间中查询k/v, key是robj类型的指针，返回的value 也是robj类型的指针👇
```javascript
/*
 * 从数据库 db 中取出键 key 的值（对象）
 *
 * 如果 key 的值存在，那么返回该值；否则，返回 NULL 。
 */
robj *lookupKey(redisDb *db, robj *key)
{
    // 查找键空间
    dictEntry *de = dictFind(db->dict, key->ptr);
    // 节点存在
    if (de)
    {
        // 取出值
        robj *val = dictGetVal(de);
		...
        // 返回值
        return val;
    }
    // 节点不存在
    return NULL;
}
```
redis的k/v空间里不是有好多种类型的数据结构吗？
如果说key都是字符串可以用一个类型来表示，
那链表，集合，有序集合，hash表呢？
我们是不是应该对不同的数据结构都写一套 存取k/v的逻辑呢？

显然是只写一套是最简单省事的哈 🐶

如果每种数据结构都来一套对应的 存取k/v逻辑，
不仅增加了开发量，而且引入了非常多的冗余逻辑，
不仅不方便人阅读，而且系统的混乱程度也会升高，
做新功能扩展一类的开发，代码会越写越屎💩

抽象出 redisObject 的好处，拿存取k/v来举例
1. 抽象层面相同逻辑的代码一套就够了。 比如存取k/v的代码
   > 针对不同数据结构来一套特定的crud代码，这个是没办法避免的事情。

   > 但是对于 k/v 在redis中的存取操作，这是可以抽象出一套代码的，
   > 因为不同数据类型的对象都可以用一个指针来指向他们，
   > 对于k/v的存取最需要关心的是 key的指针是哪个，value的指针是哪个，
   > k/v 的指针指向的结构类型是什么(类型变量，只是一个数字)，编码方式是什么。
   > 而对于 k/v指针指向的具体结构到底长啥样，对于存取k/v的指针这个操作来说 实际上是不关心的。
   > 谁要读 就 根据读出来的 指针、类型字段和实现方式 自己解析就好了。🙃️
2. 代码逻辑精简，清晰明了
便于开发者自己阅读，也便于后来的其他开发者阅读，赏心悦目。
3. 降低系统混乱程度(墒减)，系统各个部分都简单明了时，系统会比较健壮，容易扩展
因为简单嘛，所以相对复杂来说不容易出错，就比较健壮。


   > 是否在工作中有这样一种体验？ 修了一个bug，结果多了好几个bug😂
   > 这种问题一部分是因为系统太混乱，对于修改造成的影响，人在短时间之内没办法快速评估/发现所有被影响的地方。只有等上线、搞出一口大锅之后才发现原来没改对😂

   > 人脑有智慧，但接受/处理/输出 信息的速度，相比于电脑来说差太远了。
   > 金字塔原理一书中提到 人大概同一时间接收超过7个信息时，就不太能记全所有信息了。
   > 如果系统太过复杂，谨慎修bug😂
4. 因结构简单带来了 可以组合/嵌套出复杂结构的能力。
这种能力要是不通过抽象获得，无脑写代码将是一场噩梦。
   >  一个hash表里，可以存放各种类型的value, 甚至可以hash表里嵌套一个hash表。
   >  没错redis的哈希数据结构就是放在全局的k/v字典里的，这个全局k/v字典实际上就是一个hash表
5. 对象引用计数，方便对象的复用与销毁，节省空间
6. 记录对象的访问信息，以便于内存不足时进行内存淘汰

## redisObject的结构定义
redisObject一共有5个字段
1. 指向实际对象的指针
2. 实际对象的数据结构类型
3. 实际对象在一个具体数据结构类型下的编码类型
4. 引用计数
5. lru字段

结构定义如下👇
```javascript
typedef struct redisObject
{

    // 类型
    unsigned type : 4;

    // 编码
    unsigned encoding : 4;

    // 对象最后一次被访问的时间
    unsigned lru : REDIS_LRU_BITS; /* lru time (relative to server.lruclock) */

    // 引用计数
    int refcount;

    // 指向实际值的指针
    void *ptr;

} robj;
```
## redisObject 涵盖的数据类型有哪些
在redis3.0里redisObject一共有5中数据类型
```javascript
/* Object types */
// 对象类型
#define REDIS_STRING 0
#define REDIS_LIST 1
#define REDIS_SET 2
#define REDIS_ZSET 3
#define REDIS_HASH 4
```
 > 在较新一些的redis6.0里，redisObject一共有7种类型。

每一种数据类型，在redis内部都有2种及以上的 编码方式，也可以理解为实现方式。
为什么要这么搞呢？

主要是考虑到redis作为一个内存数据库，在key对应的value比较小时，使用更节省内存的方式存储。
当key对应的value元素个数或单个元素过大时，会转换成更为通用的实现方式。

在之前学习各个数据结构时，会经常发现这样的情况，再创建各个数据类型时，默认会先以节省内存的编码方式创建，当满足一定条件时，会转换为更为通用的编码方式。

所有的编码类型如下👇	
```javascript
/* Objects encoding. Some kind of objects like Strings and Hashes can be
 * internally represented in multiple ways. The 'encoding' field of the object
 * is set to one of this fields for this object. */
// 对象编码
#define REDIS_ENCODING_RAW 0        （REDIS_STRING）
#define REDIS_ENCODING_INT 1        （REDIS_STRING）
#define REDIS_ENCODING_HT 2         （REDIS_SET，REDIS_HASH）
#define REDIS_ENCODING_ZIPMAP 3     /* Encoded as zipmap */
#define REDIS_ENCODING_LINKEDLIST 4 （REDIS_LIST）
#define REDIS_ENCODING_ZIPLIST 5    （REDIS_LIST，REDIS_HASH,REDIS_ZSET）
#define REDIS_ENCODING_INTSET 6     （REDIS_SET）
#define REDIS_ENCODING_SKIPLIST 7   （REDIS_ZSET）
#define REDIS_ENCODING_EMBSTR 8     （REDIS_STRING）
```

## 数据结构类型判断
前面说，当从redis的db中存取k/v时，使用了redisObject结构，没有关心value的具体结构。
那么当value从redis的db中取出来后，再做相应逻辑之前还是要判断一下value的数据结构类型，
否则是会有问题的。

比如hget 命令对应的 hgetCommand命令，通过key从db中取出value后，判断了value是否是hash对象
源码逻辑如下👇
```javascript
void hgetCommand(redisClient *c) {
    robj *o;

    if ((o = lookupKeyReadOrReply(c,c->argv[1],shared.nullbulk)) == NULL ||
        checkType(c,o,REDIS_HASH)) return;

    // 取出并返回域的值
    addHashFieldToReply(c, o, c->argv[2]);
}
/*
 * 检查对象 o 的类型是否和 type 相同：
 *
 *  - 相同返回 0 
 *
 *  - 不相同返回 1 ，并向客户端回复一个错误
 */
int checkType(redisClient *c, robj *o, int type) {

    if (o->type != type) {
        addReply(c,shared.wrongtypeerr);
        return 1;
    }

    return 0;
}
/*
 * 为执行读取操作而从数据库中查找返回 key 的值。
 *
 * 如果 key 存在，那么返回 key 的值对象。
 *
 * 如果 key 不存在，那么向客户端发送 reply 参数中的信息，并返回 NULL 。
 */
robj *lookupKeyReadOrReply(redisClient *c, robj *key, robj *reply) {

    // 查找
    robj *o = lookupKeyRead(c->db, key);

    // 决定是否发送信息
    if (!o) addReply(c,reply);

    return o;
}
```
实际上这个 用key在redis的db中取value并判断类型的操作，在各种数据类型上都是有的

比如 取string
```javascript
void getCommand(redisClient *c) {
    getGenericCommand(c);
}
int getGenericCommand(redisClient *c) {
    robj *o;
    // 尝试从数据库中取出键 c->argv[1] 对应的值对象
    // 如果键不存在时，向客户端发送回复信息，并返回 NULL
    if ((o = lookupKeyReadOrReply(c, c->argv[1], shared.nullbulk)) == NULL) {
        return REDIS_OK;
    }
    // 值对象存在，检查它的类型
    if (o->type != REDIS_STRING) {
        // 类型错误
        addReply(c, shared.wrongtypeerr);
        return REDIS_ERR;
    }
    ...
 }
```
取list
```javascript
void lindexCommand(redisClient *c) {

    robj *o = lookupKeyReadOrReply(c,c->argv[1],shared.nullbulk);

    if (o == NULL || checkType(c,o,REDIS_LIST)) return;
    ...
}
```
其余类型...省略

将公有逻辑抽象出来，只写一套代码。不同的实现方式的具体逻辑再写不同的代码。
似乎还不错🤔

## 对象引用计数
C语言是没有GC的，自己申请的内存还得自己释放。

比如B函数通过A函数得到了一个robj*类型的变量C ，B函数用完之后理所应当的要释放掉C。
但此时问题就来了，这个C别人有没有在用呢？如果只是B自己在用那释放掉就释放了。
如果有别人在用，这个C还不能马上释放。

redis作者搞了一个简单的对象引用计数，来解决内存复用与销毁的问题。
1. 谁要"拷贝"这个redisObject，引用计数就+1
2.  谁用完了这个redisObject，引用计数就-1
引用计数为0时，free掉redisObject占用的内存空间

举个源码例子
比如判断hash表中的key是否与输入的key相等 
因为字符串类型底层有多种不同的实现方式，在比较的时候需要统一的一种形式。
getDecodedObject 会返回给定 robj的 REDIS_ENCODING_RAW 编码形式。
1. 如果robj 的编码就是 REDIS_ENCODING_RAW 那么可以复用该内存，仅增加引用计数即可
2. 否则，创建一个新的 REDIS_ENCODING_RAW编码的robj 字符串，引用计数初始为1
3. 比对完之后对 getDecodedObject 返回的结构体 减1个引用计数，若引用计数减为0，则释放该结构体
> 非字符串类型的robj会崩溃

```javascript
int dictEncObjKeyCompare(void *privdata, const void *key1,
        const void *key2)
{
    robj *o1 = (robj*) key1, *o2 = (robj*) key2;
    int cmp;

    if (o1->encoding == REDIS_ENCODING_INT &&
        o2->encoding == REDIS_ENCODING_INT)
            return o1->ptr == o2->ptr;

    o1 = getDecodedObject(o1);
    o2 = getDecodedObject(o2);
    cmp = dictSdsKeyCompare(privdata,o1->ptr,o2->ptr);
    decrRefCount(o1);
    decrRefCount(o2);
    return cmp;
}


/* Get a decoded version of an encoded object (returned as a new object).
 *
 * 以新对象的形式，返回一个输入对象的解码版本（RAW 编码）。
 *
 * If the object is already raw-encoded just increment the ref count. 
 *
 * 如果对象已经是 RAW 编码的，那么对输入对象的引用计数增一，
 * 然后返回输入对象。
 */
robj *getDecodedObject(robj *o) {
    robj *dec;

    if (sdsEncodedObject(o)) {
        incrRefCount(o);
        return o;
    }

    // 解码对象，将对象的值从整数转换为字符串
    if (o->type == REDIS_STRING && o->encoding == REDIS_ENCODING_INT) {
        char buf[32];

        ll2string(buf,32,(long)o->ptr);
        dec = createStringObject(buf,strlen(buf));
        return dec;

    } else {
        redisPanic("Unknown encoding type");
    }
}

/*
 * 为对象的引用计数增一
 */
void incrRefCount(robj *o) {
    o->refcount++;
}
/*
 * 为对象的引用计数减一
 *
 * 当对象的引用计数降为 0 时，释放对象。
 */
void decrRefCount(robj *o) {

    if (o->refcount <= 0) redisPanic("decrRefCount against refcount <= 0");

    // 释放对象
    if (o->refcount == 1) {
        switch(o->type) {
        case REDIS_STRING: freeStringObject(o); break;
        case REDIS_LIST: freeListObject(o); break;
        case REDIS_SET: freeSetObject(o); break;
        case REDIS_ZSET: freeZsetObject(o); break;
        case REDIS_HASH: freeHashObject(o); break;
        default: redisPanic("Unknown object type"); break;
        }
        zfree(o);

    // 减少计数
    } else {
        o->refcount--;
    }
}
```
## lru字段 与 内存淘汰
redisObject中的lru字段记录了k/v被访问的 时间 or 频率 相关信息。
这个lru字段 只占了24位空间

在内存不足时作为参考数据，根据淘汰策略进行数据淘汰，以释放空间。
这个redis的内存淘汰策略有多种方式，可以单独写一篇博客了，在这里先不细说了。🐶



# 小结
1. 写代码的时候最先想到的就是具体如何实现，这个想法并没有什么问题。
但这是局部视角，并不完整，缺少站在系统整体层面的思考。
所写的代码模块，在整个系统中是一个什么定位，与系统中其他模块有什么关系。
所写代码模块本身是否有更好的组织方式。
系统层面的整体思考 跟 具体功能如何实现 一样重要。(跑题了🐶)




 # 往期博客回顾
1. [redis服务器的部分启动过程](https://blog.csdn.net/a158372582/article/details/106023071)
3.  [GET命令背后的源码逻辑](https://editor.csdn.net/md/?articleId=106035658)
4. [redis的基础数据结构之 sds](https://blog.csdn.net/a158372582/article/details/106063645)
5. [redis的基础数据结构之 list](https://blog.csdn.net/a158372582/article/details/106086284)
6. [redis的基础数据结构 之 ziplist](https://blog.csdn.net/a158372582/article/details/106107759)
7. [redis 基础数据结构之 hash表](https://blog.csdn.net/a158372582/article/details/106234075)
8. [redis不稳定字典的遍历](https://blog.csdn.net/a158372582/article/details/106304649)
9. [redis 基础数据结构 之 集合](https://blog.csdn.net/a158372582/article/details/106202553)
10. [redis 基础数据结构 之 有序集合](https://blog.csdn.net/a158372582/article/details/106630445) 