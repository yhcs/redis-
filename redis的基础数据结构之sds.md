
@[TOC](redis数据结构之 简单动态字符串)
# 花絮
一位鹅厂观众老爷在我的朋友圈里评论到，他想要看一集redis集群下的GET命令处理。
在这里谢谢我远哥的捧场 😘 
写完单机redis之后 会写 redis集群相关博客
# 给新观众老爷的开场
大家好，我是弟弟！
最近读了一遍 黄健宏大佬的 **<<Redis 设计与实现>>**，对Redis 3.0版本有了一些认识，该书作者有一版添加了注释的 redis 3.0源码。

网上说Redis代码写得很好，为了加深印象和学习redis大佬的代码写作艺术，了解工作中使用的redis 命令背后的源码逻辑，便有了从redis命令角度学习redis源码的想法。
(全文提到的redis服务器，都指在 **mac os 上启动的一个默认配置的单机redis服务器**)
# 往期博客回顾
1. [redis服务器的部分启动过程](https://blog.csdn.net/a158372582/article/details/106023071)
2.  [GET命令背后的源码逻辑](https://editor.csdn.net/md/?articleId=106035658)

# redis的基础数据结构之 sds
## 说说你知道的redis数据结构
 在前两篇博客中，提到的redis源码里出现了许多的数据结构
 比如, sds, list, dict(字典), hash表，robj等...
 然后各种结构互相之间又有着说不清道不明的关系
 
 今天我们就来撸一下redis的数据结构
 按照惯例，我们从简单的说起😂
 弟弟 :"因为简单的好写啊😏"

众所周知，redis的基本数据结构有 string, linked list, set, sorted set, hash table。
说人话就是 字符串，链表，集合，有序集合，哈希表

观众老爷:"不是都众所周知了吗，不需要你讲了"
弟弟 :"给点面子，给点面子... 😅"
## redis的 sds类型
上一篇，我们愉快的发送了 `SET uid.1 我是uid.1的用户信息` 和 `get uid.1` 请求
这两个命令的key是一个 char *字符串，
redisServer.db[x].dict 这个字典里的key 应该`大部分`都是字符串
为什么我没说全部呢
因为实际存放key/value的哈希表节点是的 变量key是void *类型。
并且往db->dict 字典里添加key/value的 dictAdd函数 参数上的key也是void *类型
但是在dbAdd函数里 调用dictAdd函数 传进去的key是一个sds类型
而sds类型，其实就是char *的别名
我们可以在源码里找到相关的证据
```javascript
/* 哈希表节点 */
typedef struct dictEntry {
    void *key;    // 键
    union {
        void *val;    // 值
        uint64_t u64;
        int64_t s64;
    } v;
    // 指向下个哈希表节点，形成链表
    struct dictEntry *next;
} dictEntry;

/* 类型别名，用于指向 sdshdr 的 buf 属性 */
typedef char *sds;

/* 
...
尝试将键值对 key 和 val 添加到数据库中。
...
 */
void dbAdd(redisDb *db, robj *key, robj *val) {
	// 复制键名
    sds copy = sdsdup(key->ptr);
    // 尝试添加键值对
    int retval = dictAdd(db->dict, copy, val); 
    ...
}
/* 
...
 * 尝试将给定键值对添加到字典中
 * 只有给定键 key 不存在于字典时，添加操作才会成功
...
 */
int dictAdd(dict *d, void *key, void *val) {
	...
}
```
观众老爷:"弟弟可以，严谨，提前预防，不给自己挖坑，不给自己背锅的机会"
😏

## redis的 sdshdr 类型
这时爱学习的我不禁有思考起来，为啥搞了个别名sds呢，难道是因为....?
是啊，因为 `sdshdr`结构，这是redis大佬随手抽象出来的 `简单(`安全`)动态字符串`对象
这里的sds指向的就是 sdshdr->buf字段，
除此之外sds类型还被用来计算 sdshdr的地址，这种风骚的操作，或许只有C这类语言才能做到吧
```javascript
/* 保存字符串对象的结构 */
struct sdshdr {
    // buf 中已占用空间的长度
    int len;
    // buf 中剩余可用空间的长度
    int free;
    // 数据空间
    char buf[];
};
/* 返回 sds 实际保存的字符串的长度 */
static inline size_t sdslen(const sds s) {
    struct sdshdr *sh = (void*)(s-(sizeof(struct sdshdr)));
    return sh->len;
}
```
## sdshdr对象
观众老爷: "为啥说这个类型是一个对象呢，C还能面向对象呢？,你tmd在逗我？"
弟弟:"因为有sds.c，sds.h文件啊"
观众老爷:"有两文件就叫面向对象啦？我tm..."
弟弟:"redis大佬在sds.h文件中定义了 sds、sdshdr结构，以及操作该对象的一系列方法"
观众老爷:"嗯，这还说得过去"
弟弟:"😅虽然C不能面向对象，但是可以有一颗面向对象的心啊..."

这里又不得不说一句redis大佬🐂🍺
这个源码太长，既然redis起了个叫db的变量名，那我们就对着变量名来一套crud吧！
从crud的角度简单看几个函数
### sds对象 创建！
首先是创建一个 sds对象
```javascript
/* Create a new sds string starting from a null termined C string. */
sds sdsnew(const char *init) {
    size_t initlen = (init == NULL) ? 0 : strlen(init);
    return sdsnewlen(init, initlen);
}
sds sdsnewlen(const void *init, size_t initlen) {
    struct sdshdr *sh;
    if (init) {
        // zmalloc 不初始化所分配的内存
        sh = zmalloc(sizeof(struct sdshdr)+initlen+1);
    } else {
        // zcalloc 将分配的内存全部初始化为 0
        sh = zcalloc(sizeof(struct sdshdr)+initlen+1);
    }
    // 内存分配失败，返回
    if (sh == NULL) return NULL;
    // 设置初始化长度
    sh->len = initlen;
    // 新 sds 不预留任何空间
    sh->free = 0;
    // 如果有指定初始化内容，将它们复制到 sdshdr 的 buf 中
    if (initlen && init)
        memcpy(sh->buf, init, initlen);
    // 以 \0 结尾
    sh->buf[initlen] = '\0';
    // 返回 buf 部分，而不是整个 sdshdr
    return (char*)sh->buf;
}
```
然后是...
观众老爷:"好了好了，创建完了之后我还不会rud吗？"
弟弟:"好的，了解了解 😏"

那让我们偷个懒来贴一下 sds.h的内容
```javascript

/* 类型别名，用于指向 sdshdr 的 buf 属性 */
typedef char *sds;

/* 保存字符串对象的结构 */
struct sdshdr {
    // buf 中已占用空间的长度
    int len;
    // buf 中剩余可用空间的长度
    int free;
    // 数据空间
    char buf[];
};

/* 返回 sds 实际保存的字符串的长度 */
static inline size_t sdslen(const sds s) {
    struct sdshdr *sh = (void*)(s-(sizeof(struct sdshdr)));
    return sh->len;
}

/* 返回 sds 可用空间的长度 */
static inline size_t sdsavail(const sds s) {
    struct sdshdr *sh = (void*)(s-(sizeof(struct sdshdr)));
    return sh->free;
}

sds sdsnewlen(const void *init, size_t initlen);
sds sdsnew(const char *init);
sds sdsempty(void);
size_t sdslen(const sds s);
sds sdsdup(const sds s);
void sdsfree(sds s);
size_t sdsavail(const sds s);
sds sdsgrowzero(sds s, size_t len);
sds sdscatlen(sds s, const void *t, size_t len);
sds sdscat(sds s, const char *t);
sds sdscatsds(sds s, const sds t);
sds sdscpylen(sds s, const char *t, size_t len);
sds sdscpy(sds s, const char *t);
sds sdscatvprintf(sds s, const char *fmt, va_list ap);
#ifdef __GNUC__
sds sdscatprintf(sds s, const char *fmt, ...)
    __attribute__((format(printf, 2, 3)));
#else
sds sdscatprintf(sds s, const char *fmt, ...);
#endif
sds sdscatfmt(sds s, char const *fmt, ...);
sds sdstrim(sds s, const char *cset);
void sdsrange(sds s, int start, int end);
void sdsupdatelen(sds s);
void sdsclear(sds s);
int sdscmp(const sds s1, const sds s2);
sds *sdssplitlen(const char *s, int len, const char *sep, int seplen, int *count);
void sdsfreesplitres(sds *tokens, int count);
void sdstolower(sds s);
void sdstoupper(sds s);
sds sdsfromlonglong(long long value);
sds sdscatrepr(sds s, const char *p, size_t len);
sds *sdssplitargs(const char *line, int *argc);
sds sdsmapchars(sds s, const char *from, const char *to, size_t setlen);
sds sdsjoin(char **argv, int argc, char *sep);
/* Low level functions exposed to the user API */
sds sdsMakeRoomFor(sds s, size_t addlen);
void sdsIncrLen(sds s, int incr);
sds sdsRemoveFreeSpace(sds s);
size_t sdsAllocSize(sds s);
#endif
```
## 小结
好了又到了小结时间
在字符串上面也能玩出这么多花样，还是很🐂🍺

面向对象的好处咱就不说了，讲点容易说的好处🤔️。

因为c语言表示字符串 是的 从char *指针开始一直到'\0'结束。
这是不安全的，如果后面没有'\0'就爆炸了。

然后呢，我们对字符串是有查询它长度的需求的，总不能每次都遍历到'\0'吧，
所以搞个变量存长度是有用的，o(N) 变 o(1)
同时这个长度也暗示了，这个字符串就这么长别瞎遍历爆炸了。

对于内存来说呢，在这个字符串上一顿操作，如果你有够用的剩余空间，就不给分配内存了。
人在工作中被打断一次，再重新回到被打断前的状态，据说都要25分钟呢😱。
所以如果不是必要的话，还是不要老去申请内存，因为去申请内存就要先把手里的事情放一放啊。
所以搞个字段来记录剩余的空间也是有用的
😏真是形象生动的回答呢

好了，今天这一集就先到这里，各位观众老爷下一篇再见。

观众老爷:"弟弟今天怎么这么快"
弟弟:"是啊，周日半夜写完准备发博客，被待审核了。我差点当场去世。"
        "我又转念一想，正常人哪儿这么晚发博客的。
        这或许是csdn对我的告诫。🐶命要紧，写什么博客，早点睡觉 🙃️"
### 好吧，又被待审核了，现在是2020年5月12日 00:15:00