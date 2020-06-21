
@[TOC](redis集合的实现与 求交/并/差集)
# 给新观众老爷的开场
大家好，我是弟弟！
最近读了一遍 黄健宏大佬的 **<<Redis 设计与实现>>**，对Redis 3.0版本有了一些认识
该书作者有一版添加了注释的 redis 3.0源码
[👉官方redis的github传送门](https://github.com/antirez/redis)。
[👉黄健宏大佬添加了注释的 redis 3.0源码传送门](https://github.com/huangz1990/redis-3.0-annotated.git)


网上说Redis代码写得很好，为了加深印象和学习redis大佬的代码写作艺术，了解工作中使用的redis 命令背后的源码逻辑，便有了写博客记录学习redis源码过程的想法。


# redis集合(set)
redis 的集合是 整数类型，或者字符串 类型的无序集合，集合中的成员是唯一的。
rediis 集合有两种实现方式，这取决于集合中的对象类型。
如果集合中的对象都是 int64范围内的整数，那集合的实现方式就是整数集合，
否则集合的实现方式就是哈希表。
## redis集合实现方式 之 整数集合
在创建一个集合数据类型时，redis会先判断是否可以使用整数集合，如果可以的话，将使用整数集合。

### 整数集合结构一览
```javascript
typedef struct intset {
    // 编码方式
    uint32_t encoding;
    // 集合包含的元素数量
    uint32_t length;
    // 保存元素的数组
    int8_t contents[];
} intset;
```
- encoding字段
intset支持的编码方式
```javascript
/* Note that these encodings are ordered, so:
 * INTSET_ENC_INT16 < INTSET_ENC_INT32 < INTSET_ENC_INT64. 
 * intset 的编码方式
 */
#define INTSET_ENC_INT16 (sizeof(int16_t))
#define INTSET_ENC_INT32 (sizeof(int32_t))
#define INTSET_ENC_INT64 (sizeof(int64_t))
```
- length就是数组长度
- contents 字段
该结构中的 int8_t contents[], 仅仅作为一个数组首地址使用
存储的元素的实际类型取决于encoding
我们可以从  _intsetSet函数 (在指定整数集合的指定位置设置指定值) ,观察到这一点
```javascript
/* Set the value at pos, using the configured encoding. 
 * 根据集合的编码方式，将底层数组在 pos 位置上的值设为 value 。
 */
static void _intsetSet(intset *is, int pos, int64_t value) {
    // 取出集合的编码方式
    uint32_t encoding = intrev32ifbe(is->encoding);
    if (encoding == INTSET_ENC_INT64) {
        ((int64_t*)is->contents)[pos] = value;
        memrev64ifbe(((int64_t*)is->contents)+pos);
    } else if (encoding == INTSET_ENC_INT32) {
        ((int32_t*)is->contents)[pos] = value;
        memrev32ifbe(((int32_t*)is->contents)+pos);
    } else {
        ((int16_t*)is->contents)[pos] = value;
        memrev16ifbe(((int16_t*)is->contents)+pos);
    }
}
```
当加入整数集合的元素的变量的编码类型大于整数集合的编码类型是，将对该整数集合进行扩容。
```javascript
/* Resize the intset 
 * 调整整数集合的内存空间大小
 * 如果调整后的大小要比集合原来的大小要大，
 * 那么集合中原有元素的值不会被改变。
 * 返回值：调整大小后的整数集合
 */
static intset *intsetResize(intset *is, uint32_t len) {
    // 计算数组的空间大小
    uint32_t size = len*intrev32ifbe(is->encoding);
    // 根据空间大小，重新分配空间
    // 注意这里使用的是 zrealloc ，
    // 所以如果新空间大小比原来的空间大小要大，
    // 那么数组原有的数据会被保留
    is = zrealloc(is,sizeof(intset)+size);
    return is;
}
```
### 整数集合是怎么保证元素唯一的？
在 intsetAdd函数中时(往整数集合中添加一个元素），
会调用intsetSearch 来查找这个值应该放在哪个位置
如果找到了就直接返回了，
没找到就从pos处到数据尾 整体 后一一位，将该值设置上去

redis 整数集合中存放的值是按从小到大顺序排列的
查看 intsetSearch函数，可以看到二分查找
```javascript

/* Search for the position of "value".
 * 在集合 is 的底层数组中查找值 value 所在的索引。
 * Return 1 when the value was found and 
 * sets "pos" to the position of the value within the intset. 
 * 成功找到 value 时，函数返回 1 ，并将 *pos 的值设为 value 所在的索引。
 * Return 0 when the value is not present in the intset 
 * and sets "pos" to the position where "value" can be inserted. 
 * 当在数组中没找到 value 时，返回 0 。
 * 并将 *pos 的值设为 value 可以插入到数组中的位置。
 */
static uint8_t intsetSearch(intset *is, int64_t value, uint32_t *pos) {
    int min = 0, max = intrev32ifbe(is->length)-1, mid = -1;
    int64_t cur = -1;
    /* The value can never be found when the set is empty */
    // 处理 is 为空时的情况
    if (intrev32ifbe(is->length) == 0) {
        if (pos) *pos = 0;
        return 0;
    } else {
        /* Check for the case where we know we cannot find the value,
         * but do know the insert position. */
        // 因为底层数组是有序的，如果 value 比数组中最后一个值都要大
        // 那么 value 肯定不存在于集合中，
        // 并且应该将 value 添加到底层数组的最末端
        if (value > _intsetGet(is,intrev32ifbe(is->length)-1)) {
            if (pos) *pos = intrev32ifbe(is->length);
            return 0;
        // 因为底层数组是有序的，如果 value 比数组中最前一个值都要小
        // 那么 value 肯定不存在于集合中，
        // 并且应该将它添加到底层数组的最前端
        } else if (value < _intsetGet(is,0)) {
            if (pos) *pos = 0;
            return 0;
        }
    }
    // 在有序数组中进行二分查找
    // T = O(log N)
    while(max >= min) {
        mid = (min+max)/2;
        cur = _intsetGet(is,mid);
        if (value > cur) {
            min = mid+1;
        } else if (value < cur) {
            max = mid-1;
        } else {
            break;
        }
    }
    // 检查是否已经找到了 value
    if (value == cur) {
        if (pos) *pos = mid;
        return 1;
    } else {
        if (pos) *pos = min;
        return 0;
    }
}
```
## redis集合实现方式之 哈希表
当集合成员是一个字符串，或者是超过了int64范围的整数，或者整数集合中的成员数超过了  set_max_intset_entries (默认512) 时，redis将 使用哈希表作为集合的实现数据结构。

一个set的 成员+空值 作为一个键值对存到了哈希表里。

关于哈希表的详细信息之前博客有写过，可以点击下方链接。

[上上篇博客传送门-redis的哈希表](https://blog.csdn.net/a158372582/article/details/106234075)
[上篇博客传送门-redis不稳定哈希表的遍历](https://blog.csdn.net/a158372582/article/details/106304649)
# sadd 命令源码逻辑
通过在命令表中搜索 "SADD" 命令，可以找到对应的处理命令 saddCommand。

源码逻辑大致为
1. 从 c->db->dict 字典中，通过 集合名作为key (c->argv[1]) 取出集合对象 set。
若不存在key，将set名字作为key，创建出来的set对象作为value，加入到c->db->dict字典中。

2. 遍历参数中的set成员将其加入set对象中

👇
```javascript
void saddCommand(redisClient *c) {
    robj *set;
    int j, added = 0;
    // 取出集合对象
    set = lookupKeyWrite(c->db,c->argv[1]);
    // 对象不存在，创建一个新的，并将它关联到数据库
    if (set == NULL) {
        set = setTypeCreate(c->argv[2]);
        dbAdd(c->db,c->argv[1],set);
    // 对象存在，检查类型
    } 
    ...
    // 将所有输入元素添加到集合中
    for (j = 2; j < c->argc; j++) {
        c->argv[j] = tryObjectEncoding(c->argv[j]);
        // 只有元素未存在于集合时，才算一次成功添加
        if (setTypeAdd(set,c->argv[j])) added++;
    }
    ...
    // 返回添加元素的数量
    addReplyLongLong(c,added);
}
```
## 创建集合对象 
若key不存在时，创建集合对象会根据 第一个成员的值，来判断是否能创建整数集合，否则创建创建一个哈希表实现的set集合。
```javascript
 * 返回一个可以保存值 value 的集合。
 * 当对象的值可以被编码为整数时，返回 intset ，
 * 否则，返回普通的哈希表。
robj *setTypeCreate(robj *value) {

    if (isObjectRepresentableAsLongLong(value,NULL) == REDIS_OK)
        return createIntsetObject(); //创建一个整数集合
        
    return createSetObject();//创建一个哈希表实现的set集合
}
 * 创建一个 SET 编码的集合对象
robj *createSetObject(void) {

    dict *d = dictCreate(&setDictType,NULL);

    robj *o = createObject(REDIS_SET,d);

    o->encoding = REDIS_ENCODING_HT;

    return o;
}
/* Sets type hash table */
dictType setDictType = {
    dictEncObjHash,            /* hash function */
    NULL,                      /* key dup */
    NULL,                      /* val dup */
    dictEncObjKeyCompare,      /* key compare */
    dictRedisObjectDestructor, /* key destructor */
    NULL                       /* val destructor */
};

/*
 * 创建一个 INTSET 编码的集合对象
 */
robj *createIntsetObject(void) {
    intset *is = intsetNew();
    robj *o = createObject(REDIS_SET,is);
    o->encoding = REDIS_ENCODING_INTSET;
    return o;
}

/* 创建并返回一个新的空整数集合*/
intset *intsetNew(void) {
    // 为整数集合结构分配空间
    intset *is = zmalloc(sizeof(intset));
    // 设置初始编码
    is->encoding = intrev32ifbe(INTSET_ENC_INT16);
    // 初始化元素数量
    is->length = 0;
    return is;
}
```


# 求多个集合的交、并、差集
相信通过前面哈希表的文章各位观众老爷对集合元素的增删改查应该没有问题
那我们就来看下集合上的特色操作先看下求差集吧

## 求差集
众所周知，求差集的命令长这样，[SDIFF FIRST_KEY OTHER_KEY1..OTHER_KEYN](https://www.runoob.com/redis/sets-sdiff.html)
返回的结果就是 FIRST_KEY里 都没有在后续集合中出现过的 成员们。

通过 sdiff命令我们找到了 t_set.c/sdiffCommand 函数
先描述下该函数跟取差集相关的大致流程
1. 通过key取出各个set
2. 对各个set一顿遍历/查询，得出所求的差集

观众老爷: "😤...."

对各个set一顿遍历/查询，求差集，有两种不同的方式。
主要是考虑到不同数据量情况下，不同方式在不同条件下性能不同。
但本质都还是一样的，看看FIRST_KEY中的元素是否都在别的集合中出现过，如果没有就加入结果集。

在处理前先遍历一遍所有元素，算出以下两个值 👇
```javascript
	  long long algo_one_work = 0, algo_two_work = 0;

      // 遍历所有集合
      // sets[0]就是FIRST_KEY对应的集合，后续的依此类推
      for (j = 0; j < setnum; j++) {
          if (sets[j] == NULL) continue;

          // 计算 setnum 乘以 sets[0] 的基数之积
          algo_one_work += setTypeSize(sets[0]);
          // 计算所有集合的基数之和
          algo_two_work += setTypeSize(sets[j]);
      }
```


1. 在FIRST_KEY中的元素数量，相对其他集合较小时会采用第一种处理方式👇
```javascript
		/* Algorithm 1 has better constant times and performs less operations
        * if there are elements in common. Give it some advantage. */
       // 算法 1 的常数比较低，优先考虑算法 1
       algo_one_work /= 2;
       diff_algo = (algo_one_work <= algo_two_work) ? 1 : 2;
```
第一种处理方式:
流程如下
1. 对第二个集合开始的后面所有集合，按集合元素个数从小到大排序
2. 遍历第一个集合中每个元素，在第二个及以后的集合中查找是否存在，不存在就加入结果集 
3. 遍历完后结果集里的元素就是差集 
   > 1. 这就是为什么当第一个集合个数相对相对较小时使用这种方式，
   > 明显可以看到 这一波查找 遍历到的次数是 N*M，N是第一个集合的元素个数，M是剩余集合个数
   > 2. 其次将剩余集合按元素大小从小到大排序的意义在于，
   > 如果M里的第一个集合就包含了元素，能最快发现并跳出循环，当然数据有各种各样的反例。
   > 只能说从平均情况来看，这样是比较快的
   > （这个应该要从数学概率统计角度去给出一个完美的证明🙃️）

当第一个集合的元素的个数较大时，第一种方式不太适合。

第二种处理方式
流程如下
1. 先将第一个集合中的元素全部拿出来放入结果集。
2. 遍历剩余集合的所有元素，从结果集中删除剩余集合中的所有元素
3. 遍历完后结果集里的元素就是差集

交集/并集相信观众老爷也会求了，那今天就先到这里。求差集源码见文末

# 小结
1. 集合有两种实现方式，一种是整数集合，一种是哈希表
2. 集合的交/并/差集这个功能，可以用来做点东西
比如交集两个人的共同好友
差集推荐朋友认识的人，朋友的朋友认识的人，朋友的朋友的朋友...
并集这几个人的朋友圈里一共出现了多少个人
3. set的元素是唯一的，也比较适合用来做点赞数，统计独立ip这种事情
4. 对于集合上的求交/并/差集也是可以直接存储到一个新的集合对象的

 # 往期博客回顾
1. [redis服务器的部分启动过程](https://blog.csdn.net/a158372582/article/details/106023071)
6.  [GET命令背后的源码逻辑](https://editor.csdn.net/md/?articleId=106035658)
7. [redis的基础数据结构之 sds](https://blog.csdn.net/a158372582/article/details/106063645)
8. [redis的基础数据结构之 list](https://blog.csdn.net/a158372582/article/details/106086284)
9. [redis的基础数据结构 之 ziplist](https://blog.csdn.net/a158372582/article/details/106107759)
10. [redis 基础数据结构之 hash表](https://blog.csdn.net/a158372582/article/details/106234075)
11. [redis不稳定字典的遍历](https://blog.csdn.net/a158372582/article/details/106304649)

求差集源码👇
```javascript
void sunionDiffGenericCommand(redisClient *c, robj **setkeys, int setnum, robj *dstkey, int op) {

    // 集合数组
    robj **sets = zmalloc(sizeof(robj*)*setnum);

    setTypeIterator *si;
    robj *ele, *dstset = NULL;
    int j, cardinality = 0;
    int diff_algo = 1;

    // 取出所有集合对象，并添加到集合数组中
    for (j = 0; j < setnum; j++) {
        robj *setobj = dstkey ?
            lookupKeyWrite(c->db,setkeys[j]) :
            lookupKeyRead(c->db,setkeys[j]);

        // 不存在的集合当作 NULL 来处理
        if (!setobj) {
            sets[j] = NULL;
            continue;
        }

        // 有对象不是集合，停止执行，进行清理
        if (checkType(c,setobj,REDIS_SET)) {
            zfree(sets);
            return;
        }

        // 记录对象
        sets[j] = setobj;
    }

    /* Select what DIFF algorithm to use.
     *
     * 选择使用那个算法来执行计算
     *
     * Algorithm 1 is O(N*M) where N is the size of the element first set
     * and M the total number of sets.
     *
     * 算法 1 的复杂度为 O(N*M) ，其中 N 为第一个集合的基数，
     * 而 M 则为其他集合的数量。
     *
     * Algorithm 2 is O(N) where N is the total number of elements in all
     * the sets.
     *
     * 算法 2 的复杂度为 O(N) ，其中 N 为所有集合中的元素数量总数。
     *
     * We compute what is the best bet with the current input here. 
     *
     * 程序通过考察输入来决定使用那个算法
     */
    if (op == REDIS_OP_DIFF && sets[0]) {
        long long algo_one_work = 0, algo_two_work = 0;

        // 遍历所有集合
        for (j = 0; j < setnum; j++) {
            if (sets[j] == NULL) continue;

            // 计算 setnum 乘以 sets[0] 的基数之积
            algo_one_work += setTypeSize(sets[0]);
            // 计算所有集合的基数之和
            algo_two_work += setTypeSize(sets[j]);
        }

        /* Algorithm 1 has better constant times and performs less operations
         * if there are elements in common. Give it some advantage. */
        // 算法 1 的常数比较低，优先考虑算法 1
        algo_one_work /= 2;
        diff_algo = (algo_one_work <= algo_two_work) ? 1 : 2;

        if (diff_algo == 1 && setnum > 1) {
            /* With algorithm 1 it is better to order the sets to subtract
             * by decreasing size, so that we are more likely to find
             * duplicated elements ASAP. */
            // 如果使用的是算法 1 ，那么最好对 sets[0] 以外的其他集合进行排序
            // 这样有助于优化算法的性能
            qsort(sets+1,setnum-1,sizeof(robj*),
                qsortCompareSetsByRevCardinality);
        }
    }

    /* We need a temp set object to store our union. If the dstkey
     * is not NULL (that is, we are inside an SUNIONSTORE operation) then
     * this set object will be the resulting object to set into the target key
     *
     * 使用一个临时集合来保存结果集，如果程序执行的是 SUNIONSTORE 命令，
     * 那么这个结果将会成为将来的集合值对象。
     */
    dstset = createIntsetObject();

    // 执行的是并集计算
    if (op == REDIS_OP_UNION) {
        /* Union is trivial, just add every element of every set to the
         * temporary set. */
        // 遍历所有集合，将元素添加到结果集里就可以了
        for (j = 0; j < setnum; j++) {
            if (!sets[j]) continue; /* non existing keys are like empty sets */

            si = setTypeInitIterator(sets[j]);
            while((ele = setTypeNextObject(si)) != NULL) {
                // setTypeAdd 只在集合不存在时，才会将元素添加到集合，并返回 1 
                if (setTypeAdd(dstset,ele)) cardinality++;
                decrRefCount(ele);
            }
            setTypeReleaseIterator(si);
        }

    // 执行的是差集计算，并且使用算法 1
    } else if (op == REDIS_OP_DIFF && sets[0] && diff_algo == 1) {
        /* DIFF Algorithm 1:
         *
         * 差集算法 1 ：
         *
         * We perform the diff by iterating all the elements of the first set,
         * and only adding it to the target set if the element does not exist
         * into all the other sets.
         *
         * 程序遍历 sets[0] 集合中的所有元素，
         * 并将这个元素和其他集合的所有元素进行对比，
         * 只有这个元素不存在于其他所有集合时，
         * 才将这个元素添加到结果集。
         *
         * This way we perform at max N*M operations, where N is the size of
         * the first set, and M the number of sets. 
         *
         * 这个算法执行最多 N*M 步， N 是第一个集合的基数，
         * 而 M 是其他集合的数量。
         */
        si = setTypeInitIterator(sets[0]);
        while((ele = setTypeNextObject(si)) != NULL) {

            // 检查元素在其他集合是否存在
            for (j = 1; j < setnum; j++) {
                if (!sets[j]) continue; /* no key is an empty set. */
                if (sets[j] == sets[0]) break; /* same set! */
                if (setTypeIsMember(sets[j],ele)) break;
            }

            // 只有元素在所有其他集合中都不存在时，才将它添加到结果集中
            if (j == setnum) {
                /* There is no other set with this element. Add it. */
                setTypeAdd(dstset,ele);
                cardinality++;
            }

            decrRefCount(ele);
        }
        setTypeReleaseIterator(si);

    // 执行的是差集计算，并且使用算法 2
    } else if (op == REDIS_OP_DIFF && sets[0] && diff_algo == 2) {
        /* DIFF Algorithm 2:
         *
         * 差集算法 2 ：
         *
         * Add all the elements of the first set to the auxiliary set.
         * Then remove all the elements of all the next sets from it.
         *
         * 将 sets[0] 的所有元素都添加到结果集中，
         * 然后遍历其他所有集合，将相同的元素从结果集中删除。
         *
         * This is O(N) where N is the sum of all the elements in every set. 
         *
         * 算法复杂度为 O(N) ，N 为所有集合的基数之和。
         */
        for (j = 0; j < setnum; j++) {
            if (!sets[j]) continue; /* non existing keys are like empty sets */

            si = setTypeInitIterator(sets[j]);
            while((ele = setTypeNextObject(si)) != NULL) {
                // sets[0] 时，将所有元素添加到集合
                if (j == 0) {
                    if (setTypeAdd(dstset,ele)) cardinality++;
                // 不是 sets[0] 时，将所有集合从结果集中移除
                } else {
                    if (setTypeRemove(dstset,ele)) cardinality--;
                }
                decrRefCount(ele);
            }
            setTypeReleaseIterator(si);

            /* Exit if result set is empty as any additional removal
             * of elements will have no effect. */
            if (cardinality == 0) break;
        }
    }

    /* Output the content of the resulting set, if not in STORE mode */
    // 执行的是 SDIFF 或者 SUNION
    // 打印结果集中的所有元素
    if (!dstkey) {
        addReplyMultiBulkLen(c,cardinality);

        // 遍历并回复结果集中的元素
        si = setTypeInitIterator(dstset);
        while((ele = setTypeNextObject(si)) != NULL) {
            addReplyBulk(c,ele);
            decrRefCount(ele);
        }
        setTypeReleaseIterator(si);

        decrRefCount(dstset);

    // 执行的是 SDIFFSTORE 或者 SUNIONSTORE
    } else {
        /* If we have a target key where to store the resulting set
         * create this key with the result set inside */
        // 现删除现在可能有的 dstkey
        int deleted = dbDelete(c->db,dstkey);

        // 如果结果集不为空，将它关联到数据库中
        if (setTypeSize(dstset) > 0) {
            dbAdd(c->db,dstkey,dstset);
            // 返回结果集的基数
            addReplyLongLong(c,setTypeSize(dstset));
            notifyKeyspaceEvent(REDIS_NOTIFY_SET,
                op == REDIS_OP_UNION ? "sunionstore" : "sdiffstore",
                dstkey,c->db->id);

        // 结果集为空
        } else {
            decrRefCount(dstset);
            // 返回 0 
            addReply(c,shared.czero);
            if (deleted)
                notifyKeyspaceEvent(REDIS_NOTIFY_GENERIC,"del",
                    dstkey,c->db->id);
        }

        signalModifiedKey(c->db,dstkey);

        server.dirty++;
    }

    zfree(sets);
}
```




