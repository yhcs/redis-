
@[TOC](Redis3.0 源码阅读&实践-GET命令背后的源码逻辑)

大家好，我是弟弟！最近读了一遍 黄健宏大佬的 **<<Redis 设计与实现>>**，对Redis 3.0版本有了一些认识，该书作者有一个添加了注释的 redis 3.0源码。

网上说Redis代码写得很好，为了加深印象和学习redis大佬的代码写作艺术，了解工作中使用的redis 命令背后的源码逻辑，便有了从redis命令角度学习redis源码的想法。
(全文提到的redis服务器，都指在 **mac os 上启动的一个默认配置的单机redis服务器**)
# Redis服务器启动过程回顾
在上一篇博客中了解到，redis服务器是一个事件驱动的单线程服务器，
比如：客户端的链接请求是一个文件的读事件，除了文件事件意外，还有时间事件。

其中redis大佬对 **事件** 进行了抽象, 不管事件底层选取的是select, epoll, kqueue
都抽象出如下信息  
1. `结构体 aeApiState`  (在 kqueue.c 里包含一个 i/o对象kqfd，以及一个事件数组)
2. `函数 aeApiCreate` (在kqueue.c里, 调用 kqueue()获取fd， 并为事件数组分配空间)
3. `函数 aeApiAddEvent` (在kqueue.c里, 给kqfd设需要监听的fd，监听的事件类型(读/写))
4. `函数 aeApiPoll` (在kqueue.c里, 通过kqfd获取活跃事件并存放到 事件数组里)
5. `函数 aeApiDelEvent` (在kqueue.c里, 删除kqfd设需要监听的fd，监听的事件类型(读/写)) 
6. `函数 aeApiResize` (在kqueue.c里, 调整事件数组大小)
7. `函数 aeApiFree`  (c语言，手动释放aeApiState对象)
8. `函数 aeApiName` (选取的底层i/o模型名字)

对事件进行抽象的好处就在于，调用方可以不用关心底层实现，按抽象出来的函数写一套代码就可以了，不用ctrl c+v产生大量冗余代码。没有大量冗余代码，也使得代码逻辑清晰明了。🐂🍺

对各种事件的处理函数，是放在了全局的redisServer对象里，通过活跃事件的fd、读/写事件类型 在redisServer对象中 关联了对应的事件处理函数
 [上一篇博客传送门](https://blog.csdn.net/a158372582/article/details/106023071)
# GET命令背后的源码逻辑

## GET 使用场景
在我们愉快的发送 GET命令前，不禁想要问自己，在工作中什么情况下会用到这个 GET 命令，如果哪儿都用不到的话，我是不是可以不用学了！😃
当然一个很容易能想到的场景就是，缓存 key: id, value: id_info 这种数据的时候，再具体一点，就是通过uid查用户信息。
好了，服务器搞起来了，客户端也连上了，连发送命令的理由都想好了，那就来让我们愉快的发送GET命令吧！
## GET命令背后的源码逻辑
### 请求命令的参数处理
SET命令比GET命令稍微复杂了一点点，我们先SET一个值来GET它看看。

 1. **SET uid.1 我是uid1的用户信息**   SET命令 发送!
 2. **GET uid.1** GET命令发送!
从下面的截图我们可以看到， set命令成功
但是get命令打错了哈，不好意思。😅
观众: "没关系，原谅你再打一次"
在我们重打一次GET命令前，我们看到redisClient上提示 **(error) ERR unknown command 'ge'**
好学的我不禁思考起来，redis应该是有一个支持的命令列表吧，它才知道我们打错了命令。
观众:"emm，**弟**(这)**弟**(不)**真**(是)**聪**(废)**明**(话吗)" 
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200510150158462.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ExNTgzNzI1ODI=,size_16,color_FFFFFF,t_70)
那我们来简单看一下redis服务器收到客户端的GET命令后是一个什么样的处理流程
从上一篇我们知道处理客户端命令的函数是readQueryFromClient, 下面是缩水的源码。
可以看到的操作是， 从fd中读取请求数据到 redisClient->querybuf中,
这个querybuf是一个sds对象，不慌下面会说这个sds是啥
然后调用processInputBuffer函数进行处理，那我们接着往下看
```javascript
/*
 * 读取客户端的查询缓冲区内容
 */
 //... 代表省略x行代码
void readQueryFromClient(aeEventLoop *el, int fd, void *privdata, int mask)
{
    redisClient *c = (redisClient *)privdata;
    int nread, readlen;
    size_t qblen;
    ...
    // 读入长度（默认为 16 MB）
    readlen = REDIS_IOBUF_LEN;
	...
    qblen = sdslen(c->querybuf);
    ...
    // 为查询缓冲区分配空间
    c->querybuf = sdsMakeRoomFor(c->querybuf, readlen);
    // 读入内容到查询缓存
    nread = read(fd, c->querybuf + qblen, readlen);
	...
    // 从查询缓存重读取内容，创建参数，并执行命令
    // 函数会执行到缓存中的所有内容都被处理完为止
    processInputBuffer(c);
	...
}
```
从下面这个while循环里，我们凭感觉来感觉一下, 
c->querybuf 可能存了1条以上的客户端发来的命令，然后呢，每次处理一条命令
这两个函数 processInlineBuffer，processMultibulkBuffer 做的事情就是从querybuf中提取出一条命令的各个参数并放到 c->argv参数数组里，c->argc里放参数个数
从名字上来看 processCommand 应该是实际执行命令的函数，那我们接着往下看
```javascript

// 处理客户端输入的命令内容
void processInputBuffer(redisClient *c)
{
    while (sdslen(c->querybuf))
    {
		...
        if (c->reqtype == REDIS_REQ_INLINE)
        {
            if (processInlineBuffer(c) != REDIS_OK)
                break;
        }
        else if (c->reqtype == REDIS_REQ_MULTIBULK)
        {
            if (processMultibulkBuffer(c) != REDIS_OK)
                break;
        }
		...
        if (c->argc != 0)
        {
            // 执行命令，并重置客户端
            if (processCommand(c) == REDIS_OK)
                resetClient(c);
        }
        ...
    }
}
```
### redis命令列表
从下图可以看出，processCommand正如其命，只是**处理**命令而已，
可以看到一个 c->cmd = c->lastcmd = lookupCommand(c->argv[0]->ptr); 
c->argv[0]->ptr 就是我们发送的命令里，按空格分隔的第一个字符串，也就是我们发送的 `get`
👇下面这一顿操作，就是在查找命令
```javascript
int processCommand(redisClient *c)
{
	...
	/* Now lookup the command and check ASAP about trivial error conditions
     * such as wrong arity, bad command name and so forth. */
    // 查找命令，并进行命令合法性检查，以及命令参数个数检查
    c->cmd = c->lastcmd = lookupCommand(c->argv[0]->ptr);
    if (!c->cmd)
    {
        // 没找到指定的命令
        flagTransaction(c);
        addReplyErrorFormat(c, "unknown command '%s'",
                            (char *)c->argv[0]->ptr);
        return REDIS_OK;
    }
    ...
    // 执行命令        
    call(c, REDIS_CALL_FULL);
    ...
    return REDIS_OK;
}
```

继续看下lookupCommand 是怎么查找命令的.
server.commands 其实就是一个字典，key是名字，values是一个指针，指向对应的命令结构体
```javascript
/*
 * 根据给定命令名字（SDS），查找命令
 */
struct redisCommand *lookupCommand(sds name)
{
    return dictFetchValue(server.commands, name);
}
```
server.commands初始化的值，就是redis.c 里的全局命令数组，包含了redis支持的所有命令，
每一行就是一个命令结构体，其中包括的信息有 命令名，命令处理函数，命令参数个数 等等等..
```javascript
struct redisCommand redisCommandTable[] = {
    {"get", getCommand, 2, "r", 0, NULL, 1, 1, 1, 0, 0},
    {"set", setCommand, -3, "wm", 0, NULL, 1, 1, 1, 0, 0},
    {"setnx", setnxCommand, 3, "wm", 0, NULL, 1, 1, 1, 0, 0},
    ...省略x行代码...
   };

```
好了，到这里就弄清楚了 redis对客户端输入命令的处理 和 如何查找支持的命令。
当找不到命令时，就会返回 unknown command '%s' 信息，提示这个命令不支持

在processCommand函数里，往下看可以看到 call(c, REDIS_CALL_FULL) 这个函数
从名字上看是**执行**命令，没跑了，不得不说，redis大佬的函数命名🐂🍺 


我们再点进去看看下call函数，可以看到 执行的实际函数是 c->cmd->proc\(c\)， 这个proc是一个函数指针，指向了我们在命令列表中找到的函数。命令的执行时间也在这儿被计算出来了。
```javascript
/* Call() is the core of Redis execution of a command */
// 调用命令的实现函数，执行命令
void call(redisClient *c, int flags)
{
    // start 记录命令开始执行的时间
    long long dirty, start, duration;
    ...
    // 计算命令开始执行的时间
    start = ustime();
    // 执行实现函数
    c->cmd->proc(c)
    // 计算命令执行耗费的时间
    duration = ustime() - start;
    ...
}
```
### get命令处理函数getCommand的处理流程
从命令表上可以看到，get请求的处理函数是 getCommand， 参数个数是2个
让我们来发送一次正确的GET请求
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200510161541978.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ExNTgzNzI1ODI=,size_16,color_FFFFFF,t_70)
从上面右边的客户端我们看到正确的获取到了我们之前set的数据。
然后我们看下左边服务器，我们打印出来的调试信息
1. fd: 6 readQueryFromClient 开始处理读事件， 命令信息->  参数个数: 2, 参数:  get uid.1 
2. 查找命令列表, get命令 开始执行
3. 在db: 0 的字典中查找key: uid.1
4. 找到key: uid.1 对应的value: 我是uid.1的用户信息
5. ...

观众老爷: "等等，先解释下 db:0 是什么鬼吧，db: 0 中的字典又是个啥"
弟弟: "好的好的，没问题"。
。
1. 这个db: 0啊，实际上就是 redisServer.db数组的下标
2. 该数组元素是一个指向 redisDb结构的指针
3. 这个 redisDb里有一个 dict 字典对象，记录了该db里存放的所有的key以及value.
4. 这个 dict字典对象的实现，其实就是 地址链表法实现的 hash表 也就是dict结构里的dictht对象， 
5. 在 dictht 结构里有一个dictEntry数组，它的下标表示的是一个hash值，
6. 这个hash值使用对应类型的hash函数 (比如 MurmurHash2 ) 对key算出来的一个值 
7. 将key的hash值 做为数组下标 找到对应的hash表节点
8. 再通过遍历链表的方式，匹配具体的key，来找到key对应的value

啊，原来最后还要遍历链表啊，那会不会出现极端情况 好多的key的hash值都一样呢，那遍历链表的性能岂不是很差吗
>emmm... 这个问题可以靠优秀的hash函数对随机key产生足够分散的hash值来解决。
当字典中的简直对的个数与hash数组长度满足一定条件时，会触发 扩容/缩容，并对所有的key/value 重新进行hash映射， 一定程度上也能解决上述问题。

### 套娃一样的结构体们
好吧，让我们先看一下这一层一层套娃一样的结构 长啥样吧 😭
```javascript
/*
 * 哈希表节点
 */
typedef struct dictEntry {
    // 键
    void *key;
    // 值
    union {
        void *val;
        ...
    } v;
    // 指向下个哈希表节点，形成链表
    struct dictEntry *next;
} dictEntry;
/*
* 哈希表
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
/*
 * 字典
 */
typedef struct dict {
    ...
    // 哈希表
    dictht ht[2];
	...
} dict;

typedef struct redisDb
{
	...
    // 数据库键空间，保存着数据库中的所有键值对
    dict *dict; /* The keyspace for this DB */
    // 键的过期时间，字典的键为键，字典的值为过期事件 UNIX 时间戳
    dict *expires; /* Timeout of keys with a timeout set */
   	...
    // 数据库号码
    int id; /* Database ID */
} redisDb;

struct redisServer
{
	...
    // 数据库，db其实是数组首地址
    redisDb *db;
	...
	// db数组大小
    int dbnum;                      /* Total number of configured DBs */
  	...
};

typedef struct redisClient
{
    // 套接字描述符
    int fd;
	...
    // 当前正在使用的数据库
    redisDb *db;
	...
    // 当前正在使用的数据库的 id （号码）
    int dictid;
	...
} redisClient;
```
### get命令处理函数getCommand的源码
到这里我们了解了 getCommand函数大体的流程，以及各种套娃结构
让我们来看一眼 getCommand 函数吧
（为了控制篇幅 省略/修改了部分代码）
```javascript
void getCommand(redisClient *c)
{
    getGenericCommand(c);
}
int getGenericCommand(redisClient *c)
{
    robj *o;
    // 尝试从数据库中取出键 c->argv[1] 对应的值对象
    // 如果键不存在时，向客户端发送回复信息，并返回 NULL
    if ((o = lookupKeyReadOrReply(c, c->argv[1], shared.nullbulk)) == NULL){       
    	return REDIS_OK;
    }
    // 值对象存在，检查它的类型
    if (o->type == REDIS_STRING) {
        // 类型正确，向客户端返回对象的值
        addReplyBulk(c, o);
        return REDIS_OK;
    }
    ...
}
/*
 * 为执行读取操作而从数据库中查找返回 key 的值。
 * 如果 key 存在，那么返回 key 的值对象。
 * 如果 key 不存在，那么向客户端发送 reply 参数中的信息，并返回 NULL 。
 */
robj *lookupKeyReadOrReply(redisClient *c, robj *key, robj *reply)
{
    // 查找
    robj *o = lookupKeyRead(c->db, key);
    ...
    return o;
}
/*
 * 为执行读取操作而取出键 key 在数据库 db 中的值。
 * 并根据是否成功找到值，更新服务器的命中/不命中信息。
 * 找到时返回值对象，没找到返回 NULL 。
 */
robj *lookupKeyRead(redisDb *db, robj *key)
{
    robj *val;
    // 检查 key 释放已经过期
    expireIfNeeded(db, key);
    // 从数据库中取出键的值
    val = lookupKey(db, key)
    // 更新命中/不命中信息
    if (val == NULL)
        server.stat_keyspace_misses++;
    else
        server.stat_keyspace_hits++;
    // 返回值
    return val;
}
/*
 * 从数据库 db 中取出键 key 的值（对象）
 * 如果 key 的值存在，那么返回该值；否则，返回 NULL 。
 */
robj *lookupKey(redisDb *db, robj *key)
{
    // 查找键空间
    dictEntry *de = dictFind(db->dict, key->ptr);
    // 节点存在
    if (de) {
        // 取出值
        robj *val = dictGetVal(de);
        ...
        // 返回值
        return val;
    }
    ...
}
/*
 * 返回字典中包含键 key 的节点
 * 找到返回节点，找不到返回 NULL
 */
dictEntry *dictFind(dict *d, const void *key)
{
    dictEntry *he;
    unsigned int h, idx, table;
    // 字典（的哈希表）为空
    if (d->ht[0].size == 0) return NULL; /* We don't have a table at all */
	...
    // 计算键的哈希值
    h = dictHashKey(d, key);
    // 在字典的哈希表中查找这个键
    // T = O(1)
    for (table = 0; table <= 1; table++) {
        // 计算索引值
        idx = h & d->ht[table].sizemask;
        // 遍历给定索引上的链表的所有节点，查找 key
        he = d->ht[table].table[idx];
        // T = O(1)
        while(he) 
            if (dictCompareKeys(d, key, he->key))
                return he;
            he = he->next;
        }
		...
    }
    // 进行到这里时，说明两个哈希表都没找到
    return NULL;
}
```
好了，如果看完了上面优美的代码会发现
>redisDb->dict->ht\[2\] ，字典里有两个hash表。
>实际上 字典在进行进行 扩容/缩容操作会用到ht\[1\] 来过渡
>操作完毕之后所有的key/value对都存在ht[0]里
>这个字典 扩容/缩容的过程在后面的博客中会写

### robj结构体初现
现在我们拿到了 key对应的value
代码里可以看到value的类型是 `robj *`
emmm... 为什么不是一个char *呢，简单明了。这个robj又是啥

好吧，那我们再看下这个robj, 我们取出来的那个字符串，是放在了 void *ptr指针指向的结构里。
那这个结构体又是什么呢？
emmm.. 针对本文里的 `set uid.1 我是uid.1的用户信息` key `uid.1`对应的值，
ptr指向的是 sdshdr结构里的 char *buf 字段... 
这两结构的定义如下所示
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
struct sdshdr {
    // buf 中已占用空间的长度
    int len;
    // buf 中剩余可用空间的长度
    int free;
    // 数据空间
    char buf[];
};
```
### 回复客户端命令处理结果
上面这一顿描述，我们知道了 对于redisClient发送的GET请求
1. 服务器如何接收请求
2. 服务器如何处理请求
3. GET请求的处理函数 getCommand 的处理逻辑
4. ...处理完请求然后呢???

emmm... 让我们长话短说，get请求处理结束前，
就像服务器读客户端请求一样。

1. 服务器将要返回给客户端的值放在了 redisClient->buf 字段，
2. redisClient->bufpos 记录了buf的长度。
3. 给fd: 6设置一个写事件处理函数 sendReplyToClient
4. 将fd:6 的写事件绑定到了kqfd上。
5. 在这里fd:6上的读请求就结束了，
6. 在之后的事件循环里，fd: 6上的写事件将触发
7. sendReplyToClient被调用，
8. 将fd: 6对应的redisClient->buf的内容写入套接字
9. 然后从kqfd上删除fd: 6的写事件监听
10. fd:6 上的写事件处理结束
 
看图说话⬇️ ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200510233859352.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ExNTgzNzI1ODI=,size_16,color_FFFFFF,t_70)
好了到这里我们简单的了解了一个 redis的 get命令背后的源码逻辑。小结一下吧

# 小结

1. 读入命令并处理
```mermaid
flowchat
st=>start: redisClient发送 get uid.1
e=>end: 本轮读事件结束
op1=>operation: redisServer上在某次事件循环里
该客户端触发了一个读文件事件
op2=>operation:  readQueryFromClient函数被调用
用来处理该 get请求
op3=>operation:  提取各个参数
op4=>operation:  在redisServer的全局命令表中找到处理get请求的函数getCommand
op5=>operation:  getCommand命令被调用
op6=>operation:  redisClient默认对应的db是0
在redisServer的0号db中的dict字典里查找key: uid.1
op7=>operation:  找到key对应的value，并写入redisClient的buf字段
op8=>operation:  给该客户端设置写事件处理函数sendReplyToClient
并设置kqfd监听该客户端fd的写事件
st->op1->op2->op3->op4->op5->op6->op7->op8->e
```
2. 将命令结果返回客户端
```mermaid
flowchat
st=>start: redisServer上在某次事件循环里
触发了该客户端一个写文件事件
e=>end: 本轮写事件结束
op1=>operation:  sendReplyToClient函数被调用
op2=>operation:  将redisClient的buf字段指向的内容写入套接字fd: 6
op3=>operation:  将fd: 6上的写事件监听从kqfd上摘除

st->op1->op2->op3->e
```

...
好了，redis的get命令是完事了。

但是不是觉得哪里不对劲
观众:"那么多一层一层套娃的结构体是tm怎么回事，看得我眼睛都花了"
弟弟:"了解了，那下一篇就先捋一捋套娃一样的结构体吧😪"