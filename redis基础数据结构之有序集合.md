
@[TOC](redis有序集合的实现 以及 zrank-zadd-zrange的源码逻辑)
# 给新观众老爷的开场
大家好，我是弟弟！
最近读了一遍 黄健宏大佬的 **<<Redis 设计与实现>>**，对Redis 3.0版本有了一些认识
该书作者有一版添加了注释的 redis 3.0源码
[👉官方redis的github传送门](https://github.com/antirez/redis)。
[👉黄健宏大佬添加了注释的 redis 3.0源码传送门](https://github.com/huangz1990/redis-3.0-annotated.git)


网上说Redis代码写得很好，为了加深印象和学习redis大佬的代码写作艺术，了解工作中使用的redis 命令背后的源码逻辑，便有了写博客记录学习redis源码过程的想法。


# redis 有序集合(zset)
redis的有序集合 跟 集合一样成员是唯一的，跟集合不同的是每个成员都会关联一个分数用来排序，分数可以重复。

redis有序集合的底层实现数据结构 有两种，分别是  ziplist和zset。
> 这取决于有序集合中的成员数量或者 单个成员的key的长度

当有序集合成员数超过 zset_max_ziplist_entries(默认128),
 或者成员的key的长度超过 zset_max_ziplist_value(默认64)时，
 会将有序集合的实现方式从ziplist转化为zset。
 在其他一些地方, 也可能从zset转化成ziplist。

## redis有序集合 第一种实现方式 ziplist
ziplist实现的有序集合，会将成员的key和分数拆成两个紧挨着的ziplist元素，key在前，分数灾后。
有序集合成员按 成员分数从小到大排列，
相同分数的有序集合成员，按照成员key的字典序从小到大排列。

ziplist的详细内容可以参考往期博客 👇
[往期博客 redis源码阅读 - 基础数据结构 之 ziplist](https://blog.csdn.net/a158372582/article/details/106107759)

## redis有序集合 第二种实现方式 zset
zset实现的有序集合，包含了一个 k/v字典 和 一个 跳跃表

zset结构一览 👇
```javascript
 * 有序集合
typedef struct zset
{
    // 字典，键为成员，值为分值
    // 用于支持 O(1) 复杂度的按成员取分值操作
    dict *dict;
    // 跳跃表，按分值排序成员
    // 用于支持平均复杂度为 O(log N) 的按分值定位成员操作
    // 以及范围操作
    zskiplist *zsl;
} zset;
```
哈希表的详细结构可参考往期博客
[redis 基础数据结构之 hash表](https://blog.csdn.net/a158372582/article/details/106234075)
[redis不稳定字典的遍历](https://blog.csdn.net/a158372582/article/details/106304649)
### 跳跃表
跳跃表可以简单理解为 是一个用链表实现的 
使用二分查找思想来加速查询，具有区间遍历功能的数据结构

跳跃表里有一个双向链表，便于双向遍历。每个链表元素记录了成员的key和分数
并且链表元素的顺序是按分值从小到大排列，相同分值按key的字典序从小到大排列。

在每一个链表节点上，有一个层的概念，
在最大层数以下，每一层有两个字段，
一个是该层的跨度，也就是在该层从当前节点n1,到节点n2 中间跨越了几个元素
层越高，跨度越大
另一个就是指向节点n2的指针，叫当前层的前进指针
每个链表节点还有一个指向前一个链表的指针，叫后退指针

这样不管是计算链表节点的排名，还是做区间遍历，都能快速定位到需要查找的节点
> 因为层越高跨度越大，从高层开始遍历查找是，能经过很少的比较次数，就能快速定位到所要查找节点，在该层所在的区间，然后再一层一层遍历下去，直到找到


跳跃表结构 与跳跃表节点结构一览
```javascript
/*
 * 跳跃表
 */
typedef struct zskiplist
{
    // 表头节点和表尾节点
    struct zskiplistNode *header, *tail;
    // 表中节点的数量
    unsigned long length;
    // 表中层数最大的节点的层数
    int level;
} zskiplist;


/* ZSETs use a specialized version of Skiplists */
/*
 * 跳跃表节点
 */
typedef struct zskiplistNode
{
    // 成员对象
    robj *obj;
    // 分值
    double score;
    // 后退指针,指向前一个跳跃表节点
    struct zskiplistNode *backward;
    // 层
    struct zskiplistLevel
    {
        // 前进指针，指向跨越span个元素后的跳跃表节点
        struct zskiplistNode *forward;
        // 跨度
        unsigned int span;
    } level[];
} zskiplistNode;
```




### 在跳跃表上查找元素
在zset中单纯的查找key的跳跃表节点，是通过字典，也就是hash表的查找来完成的
当在跳跃表上插入一个跳跃表节点，或者取跳跃表节点的排名等操作时，需要在跳跃表上进行。
> 查找单个元素，hash表的查询效率一般是o(1)的，而跳跃表一般是o(logN)

### ZRANK 在跳跃表上取节点的排名

zrank跳跃表相关的处理逻辑大致为
1. 通过有序集合的key，找到有序集合(仅讨论zset)
2. 通过成员名在zset的字典中查找，对应的key是否存在
3. 存在，则返回字典中的value，也就是成员的分数
4. 通过成员key和成员分数，遍历跳跃表计算出排名

skiplist计算成员排名的逻辑大致为
1. 从跳跃表的头节点的最高层开始遍历
6. 在每一层中，通过成员分数以及成员的key，来查找所在的区间
7. 查找区间的方法为，
当前层的前进指针指向的节点的分数

   3.1   
   若小于被查找成员分数，若分数相等，且key小于被查找成员的key
则在当前层中向前遍历，累加当前节点的跨度记录到rank变量中，
并将指针设为前进指针


   3.2 
   否则,被查找成员落在该区域，层数减一，继续遍历
8. 结束条件
4.1 找到被查找成员,返回rank值
4.2 层高小于0，没查到，返回0 
   > skiplist的rank值是从1开始的，因为有一个表头节点，详细在后面sadd相关的代码逻辑里能看到
   

让我们来看下zrank取排名的源码逻辑，zrank命令对应的处理函数是
```javascript
void zrankCommand(redisClient *c) {
    zrankGenericCommand(c, 0);
}

void zrankGenericCommand(redisClient *c, int reverse) {
    robj *key = c->argv[1];
    robj *ele = c->argv[2];
    robj *zobj;
    unsigned long llen;
    unsigned long rank;

    // 有序集合
    if ((zobj = lookupKeyReadOrReply(c,key,shared.nullbulk)) == NULL ||
        checkType(c,zobj,REDIS_ZSET)) return;
    // 元素数量
    llen = zsetLength(zobj);
	...
    if (zobj->encoding == REDIS_ENCODING_ZIPLIST) {//压缩列表
        ...
    } else if (zobj->encoding == REDIS_ENCODING_SKIPLIST) {//跳跃表
        zset *zs = zobj->ptr;
        zskiplist *zsl = zs->zsl;
        dictEntry *de;
        double score;

        // 从字典中取出元素
        ele = c->argv[2] = tryObjectEncoding(c->argv[2]);
        de = dictFind(zs->dict,ele);
        if (de != NULL) {

            // 取出元素的分值
            score = *(double*)dictGetVal(de);

            // 在跳跃表中计算该元素的排位
            rank = zslGetRank(zsl,score,ele);
            redisAssertWithInfo(c,ele,rank); /* Existing elements always have a rank. */

            // ZRANK 还是 ZREVRANK ？
            if (reverse)
                addReplyLongLong(c,llen-rank);
            else
                addReplyLongLong(c,rank-1);
        } else {
            addReply(c,shared.nullbulk);
        }

    } else {
        redisPanic("Unknown sorted set encoding");
    }
}

unsigned int zsetLength(robj *zobj) {

    int length = -1;

    if (zobj->encoding == REDIS_ENCODING_ZIPLIST) {
        length = zzlLength(zobj->ptr);

    } else if (zobj->encoding == REDIS_ENCODING_SKIPLIST) {
        length = ((zset*)zobj->ptr)->zsl->length;

    } else {
        redisPanic("Unknown sorted set encoding");
    }

    return length;
}

/* Find the rank for an element by both score and key.
 *
 * 查找包含给定分值和成员对象的节点在跳跃表中的排位。
 *
 * Returns 0 when the element cannot be found, rank otherwise.
 *
 * 如果没有包含给定分值和成员对象的节点，返回 0 ，否则返回排位。
 *
 * Note that the rank is 1-based due to the span of zsl->header to the
 * first element. 
 *
 * 注意，因为跳跃表的表头也被计算在内，所以返回的排位以 1 为起始值。
 *
 * T_wrost = O(N), T_avg = O(log N)
 */
unsigned long zslGetRank(zskiplist *zsl, double score, robj *o) {
    zskiplistNode *x;
    unsigned long rank = 0;
    int i;

    // 遍历整个跳跃表
    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {

        // 遍历节点并对比元素
        while (x->level[i].forward &&
            (x->level[i].forward->score < score ||
                // 比对分值
                (x->level[i].forward->score == score &&
                // 比对成员对象
                compareStringObjects(x->level[i].forward->obj,o) <= 0))) {

            // 累积跨越的节点数量
            rank += x->level[i].span;

            // 沿着前进指针遍历跳跃表
            x = x->level[i].forward;
        }

        /* x might be equal to zsl->header, so test if obj is non-NULL */
        // 必须确保不仅分值相等，而且成员对象也要相等
        // T = O(N)
        if (x->obj && equalStringObjects(x->obj,o)) {
            return rank;
        }
    }

    // 没找到
    return 0;
}
```
### ZADD 在跳跃表中加入一个成员(key+分数)
在跳跃表中插入一个成员的逻辑大概如下
1. 首先要找到在何处插入成员，查找方法跟zrank查找的方法一样
2. 该成员从最高层 maxlevel 到最底层 minlevel，每一层都存在一个该分数对应的区间，
   >  这点没有问题吧，可以想想为什么🙃️

    遍历过程中会记录 相应区间左边的那个节点 记在update[level]中，
因为后续会用来更新部分受影响的跳跃表的区间

4. 在该成员被插入后，会给该成员随机产生一个层数 newlevel，
   > 若newlevel > maxlevel
将跳跃表表头节点的层数从 maxlevel 升到 newlevel ，
并且将 maxlevel 升到 newlevel 的每一层的 span设置为当前跳跃表元素的个数
并记录每层需要被更新的节点 update[level] = zsl->header 跳跃表表头节点

   > newlevel的随机生成机制，让越大的层数产生的几率越小。
这样让跳跃表在高层具有较大的区间跨度，从高层往底层，区间跨度相对越来越小，用来加速查找。
    
4. 从最底层minlevel,到newlevel，将每一层的update[level]修正
4.1  
每层包含了新插入节点的区间会被一分为二
  也就是 update[level] 到 update[level].forward 这一个区间会被一份为二
  
    4.2 	
    并且会修正 minlevel 到 newlevel每层update[level]节点的span值，因为区间被一分为二，span值可能会受影响。
    若newlevel小于maxlevel,将newlevel到maxlevel的update[level]的span值+1
  
    4.3 
    并且会修正minlevel 到 newlevel 每层update[level], update[level].forward,以及新节点的 前/后指针
  
	  > 如果没有这些骚操作，往跳跃表中插入了一个成员会怎样？
	  🙃️   那不就成裸的链表了吗

来看下在跳跃表中插入一个成员的源码逻辑
```javascript
/*
 * 创建一个成员为 obj ，分值为 score 的新节点，
 * 并将这个新节点插入到跳跃表 zsl 中。
 * 
 * 函数的返回值为新节点。
 *
 * T_wrost = O(N^2), T_avg = O(N log N)
 */
#define ZSKIPLIST_MAXLEVEL 32 /* Should be enough for 2^32 elements */
zskiplistNode *zslInsert(zskiplist *zsl, double score, robj *obj) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    unsigned int rank[ZSKIPLIST_MAXLEVEL];
    int i, level;

    redisAssert(!isnan(score));

    // 在各个层查找节点的插入位置
    // T_wrost = O(N^2), T_avg = O(N log N)
    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {

        /* store rank that is crossed to reach the insert position */
        // 如果 i 不是 zsl->level-1 层
        // 那么 i 层的起始 rank 值为 i+1 层的 rank 值
        // 各个层的 rank 值一层层累积
        // 最终 rank[0] 的值加一就是新节点的前置节点的排位
        // rank[0] 会在后面成为计算 span 值和 rank 值的基础
        rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];

        // 沿着前进指针遍历跳跃表
        // T_wrost = O(N^2), T_avg = O(N log N)
        while (x->level[i].forward &&
            (x->level[i].forward->score < score ||
                // 比对分值
                (x->level[i].forward->score == score &&
                // 比对成员， T = O(N)
                compareStringObjects(x->level[i].forward->obj,obj) < 0))) {

            // 记录沿途跨越了多少个节点
            rank[i] += x->level[i].span;

            // 移动至下一指针
            x = x->level[i].forward;
        }
        // 记录将要和新节点相连接的节点
        update[i] = x;
    }

    /* we assume the key is not already inside, since we allow duplicated
     * scores, and the re-insertion of score and redis object should never
     * happen since the caller of zslInsert() should test in the hash table
     * if the element is already inside or not. 
     *
     * zslInsert() 的调用者会确保同分值且同成员的元素不会出现，
     * 所以这里不需要进一步进行检查，可以直接创建新元素。
     */

    // 获取一个随机值作为新节点的层数
    // T = O(N)
    level = zslRandomLevel();

    // 如果新节点的层数比表中其他节点的层数都要大
    // 那么初始化表头节点中未使用的层，并将它们记录到 update 数组中
    // 将来也指向新节点
    if (level > zsl->level) {

        // 初始化未使用层
        // T = O(1)
        for (i = zsl->level; i < level; i++) {
            rank[i] = 0;
            update[i] = zsl->header;
            update[i]->level[i].span = zsl->length;
        }

        // 更新表中节点最大层数
        zsl->level = level;
    }

    // 创建新节点
    x = zslCreateNode(level,score,obj);

    // 将前面记录的指针指向新节点，并做相应的设置
    // T = O(1)
    for (i = 0; i < level; i++) {
        
        // 设置新节点的 forward 指针
        x->level[i].forward = update[i]->level[i].forward;
        
        // 将沿途记录的各个节点的 forward 指针指向新节点
        update[i]->level[i].forward = x;

        /* update span covered by update[i] as x is inserted here */
        // 计算新节点跨越的节点数量
        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);

        // 更新新节点插入之后，沿途节点的 span 值
        // 其中的 +1 计算的是新节点
        update[i]->level[i].span = (rank[0] - rank[i]) + 1;
    }

    /* increment span for untouched levels */
    // 未接触的节点的 span 值也需要增一，这些节点直接从表头指向新节点
    // T = O(1)
    for (i = level; i < zsl->level; i++) {
        update[i]->level[i].span++;
    }

    // 设置新节点的后退指针
    x->backward = (update[0] == zsl->header) ? NULL : update[0];
    if (x->level[0].forward)
        x->level[0].forward->backward = x;
    else
        zsl->tail = x;

    // 跳跃表的节点计数增一
    zsl->length++;

    return x;
}

/* Returns a random level for the new skiplist node we are going to create.
 *
 * 返回一个随机值，用作新跳跃表节点的层数。
 *
 * The return value of this function is between 1 and ZSKIPLIST_MAXLEVEL
 * (both inclusive), with a powerlaw-alike distribution where higher
 * levels are less likely to be returned. 
 *
 * 返回值介乎 1 和 ZSKIPLIST_MAXLEVEL 之间（包含 ZSKIPLIST_MAXLEVEL），
 * 根据随机算法所使用的幂次定律，越大的值生成的几率越小。
 *
 * T = O(N)
 */
#define ZSKIPLIST_P 0.25      /* Skiplist P = 1/4 */
int zslRandomLevel(void) {
    int level = 1;

    while ((random()&0xFFFF) < (ZSKIPLIST_P * 0xFFFF))
        level += 1;

    return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
}


/*
 * 创建一个层数为 level 的跳跃表节点，
 * 并将节点的成员对象设置为 obj ，分值设置为 score 。
 *
 * 返回值为新创建的跳跃表节点
 *
 * T = O(1)
 */
zskiplistNode *zslCreateNode(int level, double score, robj *obj) {
    
    // 分配空间
    zskiplistNode *zn = zmalloc(sizeof(*zn)+level*sizeof(struct zskiplistLevel));

    // 设置属性
    zn->score = score;
    zn->obj = obj;

    return zn;
}
```
到这里我们就了解到了在跳跃表中取排名，已经插入成员的源码逻辑。

zadd命令就是对这些功能的包装函数
若zadd添加的是一个不存在的成员，会在跳跃表中插入该成员节点
若是已存在的节点，会先删除该成员节点，再重新插入该成员节点。
删除成员节点的逻辑，差不多就是插入的逻辑的逆过程

来让我们看下zadd与跳跃表相关的源码逻辑👇

```javascript
void zaddCommand(redisClient *c) {
    zaddGenericCommand(c,0);
}

/* This generic command implements both ZADD and ZINCRBY. */
void zaddGenericCommand(redisClient *c, int incr) {

    static char *nanerr = "resulting score is not a number (NaN)";

    robj *key = c->argv[1];
    robj *ele;
    robj *zobj;
    robj *curobj;
    double score = 0, *scores = NULL, curscore = 0.0;
    int j, elements = (c->argc-2)/2;
    int added = 0, updated = 0;

    // 输入的 score - member 参数必须是成对出现的
    if (c->argc % 2) {
        addReply(c,shared.syntaxerr);
        return;
    }

    /* Start parsing all the scores, we need to emit any syntax error
     * before executing additions to the sorted set, as the command should
     * either execute fully or nothing at all. */
    // 取出所有输入的 score 分值
    scores = zmalloc(sizeof(double)*elements);
    for (j = 0; j < elements; j++) {
        if (getDoubleFromObjectOrReply(c,c->argv[2+j*2],&scores[j],NULL)
            != REDIS_OK) goto cleanup;
    }

    /* Lookup the key and create the sorted set if does not exist. */
    // 取出有序集合对象
    zobj = lookupKeyWrite(c->db,key);
    if (zobj == NULL) {
        // 有序集合不存在，创建新有序集合
        if (server.zset_max_ziplist_entries == 0 ||
            server.zset_max_ziplist_value < sdslen(c->argv[3]->ptr))
        {
            zobj = createZsetObject();
        } else {
            zobj = createZsetZiplistObject();
        }
        // 关联对象到数据库
        dbAdd(c->db,key,zobj);
    } else {
        // 对象存在，检查类型
        if (zobj->type != REDIS_ZSET) {
            addReply(c,shared.wrongtypeerr);
            goto cleanup;
        }
    }

    // 处理所有元素
    for (j = 0; j < elements; j++) {
        score = scores[j];

        // 有序集合为 ziplist 编码
        if (zobj->encoding == REDIS_ENCODING_ZIPLIST) {
           ...
        // 有序集合为 SKIPLIST 编码
        } else if (zobj->encoding == REDIS_ENCODING_SKIPLIST) {
            zset *zs = zobj->ptr;
            zskiplistNode *znode;
            dictEntry *de;

            // 编码对象
            ele = c->argv[3+j*2] = tryObjectEncoding(c->argv[3+j*2]);

            // 查看成员是否存在
            de = dictFind(zs->dict,ele);
            if (de != NULL) {

                // 成员存在

                // 取出成员
                curobj = dictGetKey(de);
                // 取出分值
                curscore = *(double*)dictGetVal(de);

                // ZINCRYBY 时执行
                ...
                
                /* Remove and re-insert when score changed. We can safely
                 * delete the key object from the skiplist, since the
                 * dictionary still has a reference to it. */
                // 执行 ZINCRYBY 命令时，
                // 或者用户通过 ZADD 修改成员的分值时执行
                if (score != curscore) {
                    // 删除原有元素
                    redisAssertWithInfo(c,curobj,zslDelete(zs->zsl,curscore,curobj));

                    // 重新插入元素
                    znode = zslInsert(zs->zsl,score,curobj);
                    incrRefCount(curobj); /* Re-inserted in skiplist. */

                    // 更新字典的分值指针
                    dictGetVal(de) = &znode->score; /* Update score ptr. */

                    server.dirty++;
                    updated++;
                }
            } else {

                // 元素不存在，直接添加到跳跃表
                znode = zslInsert(zs->zsl,score,ele);
                incrRefCount(ele); /* Inserted in skiplist. */

                // 将元素关联到字典
                redisAssertWithInfo(c,NULL,dictAdd(zs->dict,ele,&znode->score) == DICT_OK);
                incrRefCount(ele); /* Added to dictionary. */

                server.dirty++;
                added++;
            }
        } else {
            redisPanic("Unknown sorted set encoding");
        }
    }

    if (incr) /* ZINCRBY */
        addReplyDouble(c,score);
    else /* ZADD */
        addReplyLongLong(c,added);

cleanup:
   ...
}

/*
 * 创建一个 SKIPLIST 编码的有序集合
 */
robj *createZsetObject(void) {

    zset *zs = zmalloc(sizeof(*zs));

    robj *o;

    zs->dict = dictCreate(&zsetDictType,NULL);
    zs->zsl = zslCreate();

    o = createObject(REDIS_ZSET,zs);

    o->encoding = REDIS_ENCODING_SKIPLIST;

    return o;
}

/*
 * 创建并返回一个新的跳跃表
 *
 * T = O(1)
 */
zskiplist *zslCreate(void) {
    int j;
    zskiplist *zsl;

    // 分配空间
    zsl = zmalloc(sizeof(*zsl));

    // 设置高度和起始层数
    zsl->level = 1;
    zsl->length = 0;

    // 初始化表头节点
    // T = O(1)
    zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL,0,NULL);
    for (j = 0; j < ZSKIPLIST_MAXLEVEL; j++) {
        zsl->header->level[j].forward = NULL;
        zsl->header->level[j].span = 0;
    }
    zsl->header->backward = NULL;

    // 设置表尾
    zsl->tail = NULL;

    return zsl;
}

zskiplistNode *zslCreateNode(int level, double score, robj *obj) {
    
    // 分配空间
    zskiplistNode *zn = zmalloc(sizeof(*zn)+level*sizeof(struct zskiplistLevel));

    // 设置属性
    zn->score = score;
    zn->obj = obj;

    return zn;
}
```


### ZRANGE 取top N到 top M之间的成员
#### 为什么redis选择跳跃表而不是平衡二叉树来实现有序集合
平衡二叉树中有代表性的如 AVL树，红黑树。
跳跃表相比于红黑树这种数据结构，众所周知的优点就是，crud性能可以媲美红黑树，
数据结构简单，实现难度大大小于红黑树。
那么除了跳跃表简单好实现以外，还有什么别的特殊的原因让redis选择了跳跃表而不是红黑树？

对于这个问题的解答，如果redis大佬已经回答过的话，那直接看他怎么回答是最合适的了😂
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020060900225617.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ExNTgzNzI1ODI=,size_16,color_FFFFFF,t_70)
[原文链接](https://news.ycombinator.com/item?id=1171423)
redis的有序集合上经常有区间遍历操作。通过跳跃表加速查找到区间头，然后在链表上进行区间遍历，性能和平衡二叉树差不多。
> 疑问? 平衡二叉树似乎也能按中序遍历的顺序，将各个节点连接起来，形成一条有序的链表，似乎也可行。
> 但是这种为了遍历而串成的链表，似乎没有跳跃表的链表来的自然 🤔

#### ZRANGE 与跳跃表相关的源码逻辑
源码逻辑大概如下
1. 一段操作，检查参数，修正参数，算出start, end
1. 通过跳跃表加速查找到 start节点的位置
2. 从start 节点开始，遍历链表

源码如下👇
```javascript
void zrangeCommand(redisClient *c) {
    zrangeGenericCommand(c,0);
}


void zrangeGenericCommand(redisClient *c, int reverse) {
    robj *key = c->argv[1];
    robj *zobj;
    int withscores = 0;
    long start;
    long end;
    int llen;
    int rangelen;

    // 取出 start 和 end 参数
    if ((getLongFromObjectOrReply(c, c->argv[2], &start, NULL) != REDIS_OK) ||
        (getLongFromObjectOrReply(c, c->argv[3], &end, NULL) != REDIS_OK)) return;

    // 确定是否显示分值
    if (c->argc == 5 && !strcasecmp(c->argv[4]->ptr,"withscores")) {
        withscores = 1;
    } else if (c->argc >= 5) {
        addReply(c,shared.syntaxerr);
        return;
    }

    // 取出有序集合对象
    if ((zobj = lookupKeyReadOrReply(c,key,shared.emptymultibulk)) == NULL
         || checkType(c,zobj,REDIS_ZSET)) return;

    /* Sanitize indexes. */
    // 将负数索引转换为正数索引
    llen = zsetLength(zobj);
    if (start < 0) start = llen+start;
    if (end < 0) end = llen+end;
    if (start < 0) start = 0;

    /* Invariant: start >= 0, so this test will be true when end < 0.
     * The range is empty when start > end or start >= length. */
    // 过滤/调整索引
    if (start > end || start >= llen) {
        addReply(c,shared.emptymultibulk);
        return;
    }
    if (end >= llen) end = llen-1;
    rangelen = (end-start)+1;

    /* Return the result in form of a multi-bulk reply */
    addReplyMultiBulkLen(c, withscores ? (rangelen*2) : rangelen);

    if (zobj->encoding == REDIS_ENCODING_ZIPLIST) {
       ...

    } else if (zobj->encoding == REDIS_ENCODING_SKIPLIST) {
        zset *zs = zobj->ptr;
        zskiplist *zsl = zs->zsl;
        zskiplistNode *ln;
        robj *ele;

        /* Check if starting point is trivial, before doing log(N) lookup. */
        // 迭代的方向
        if (reverse) {
            ln = zsl->tail;
            if (start > 0)
                ln = zslGetElementByRank(zsl,llen-start);
        } else {
            ln = zsl->header->level[0].forward;
            if (start > 0)
                ln = zslGetElementByRank(zsl,start+1);
        }

        // 取出元素
        while(rangelen--) {
            redisAssertWithInfo(c,zobj,ln != NULL);
            ele = ln->obj;
            addReplyBulk(c,ele);
            if (withscores)
                addReplyDouble(c,ln->score);
            ln = reverse ? ln->backward : ln->level[0].forward;
        }
    } else {
        redisPanic("Unknown sorted set encoding");
    }
}
```


# 小结
1. 有序集合中，成员是唯一的，但分数可以重复
2. 有序集合中按分数从小到大，当分数相同按成员字典顺序从小到大排列
3. 有序集合中有一个双向链表，所以可以双向遍历
4. 有序集合适合用来做排行榜之类的功能，当然如果单个有序集合成员数过多，占用的内存也会很大。
5. 有序集合在3.0版本中，最高32层

 # 往期博客回顾
1. [redis服务器的部分启动过程](https://blog.csdn.net/a158372582/article/details/106023071)
7.  [GET命令背后的源码逻辑](https://editor.csdn.net/md/?articleId=106035658)
8. [redis的基础数据结构之 sds](https://blog.csdn.net/a158372582/article/details/106063645)
9. [redis的基础数据结构之 list](https://blog.csdn.net/a158372582/article/details/106086284)
10. [redis的基础数据结构 之 ziplist](https://blog.csdn.net/a158372582/article/details/106107759)
11. [redis 基础数据结构之 hash表](https://blog.csdn.net/a158372582/article/details/106234075)
12. [redis不稳定字典的遍历](https://blog.csdn.net/a158372582/article/details/106304649)
13. [redis集合的实现与 求交/并/差集](https://blog.csdn.net/a158372582/article/details/106202553)