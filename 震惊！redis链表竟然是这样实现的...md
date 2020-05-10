
@[TOC](redis数据结构之 链表)
# 给新观众老爷的开场
大家好，我是弟弟！
最近读了一遍 黄健宏大佬的 **<<Redis 设计与实现>>**，对Redis 3.0版本有了一些认识，该书作者有一版添加了注释的 redis 3.0源码。

网上说Redis代码写得很好，为了加深印象和学习redis大佬的代码写作艺术，了解工作中使用的redis 命令背后的源码逻辑，便有了从redis命令角度学习redis源码的想法。
(全文提到的redis服务器，都指在 **mac os 上启动的一个默认配置的单机redis服务器**)

# redis的基础数据结构之 链表
## 我以为的链表是这样的👇
```javascript
typedef struct listNode {
    // 前置节点
    struct listNode *prev;
    // 后置节点
    struct listNode *next;
    // value的类型取决于具体的场景
    int value;
} listNode;
```
## redis中的链表是这样的👇
```javascript
 /*
 * 双端链表结构
 */
typedef struct list {
    // 表头节点
    listNode *head;
    // 表尾节点
    listNode *tail;
    // 节点值复制函数
    void *(*dup)(void *ptr);
    // 节点值释放函数
    void (*free)(void *ptr);
    // 节点值对比函数
    int (*match)(void *ptr, void *key);
    // 链表所包含的节点数量
    unsigned long len;
} list;
/*
 * 双端链表节点
 */
typedef struct listNode {
    // 前置节点
    struct listNode *prev;
    // 后置节点
    struct listNode *next;
    // 节点的值
    void *value;
} listNode;
/*
 * 双端链表迭代器
 */
typedef struct listIter {
    // 当前迭代到的节点
    listNode *next;
    // 迭代的方向
    int direction;
} listIter;
```

观众老爷:"代码看着费劲，来，开始你的表演"
弟弟 :“ 😅 ”

## Q1: 为什么redis链表节点的value字段是void *类型
个人理解，这个体现 redis链表的高度抽象(对，又是抽象)。
链表就做链表的事情，把你们一个一个节点链接起来，有前/后指针就完事。
至于值，想放啥放啥。当然了，对于链表节点里的值怎么解读，跟链表没关系。
谁放的值谁负责解读 🙃️

这样一个链表里就能放任意数据类型了，让我们在源码里找一找证据。

### redisServer里的 list *Client与 list *slowlog
1. 当一个redis客户端连接上redis服务器后，会创建一个redisClient，并且该对象的指针被加入到了 redisServer->Clients
2. 当一个命令被执行完毕，如果命令执行慢，将慢日志写入redisServer->slowlog里

redisServer->Clients 与 redisServer->slowlog 都是list *类型，源码如下👇

```javascript
struct redisServer
{
	...
    // 一个链表，保存了所有客户端状态结构
    list *clients; /* List of active clients */
    ...
    // 保存了所有慢查询日志的链表
    list *slowlog; /* SLOWLOG list of commands */
	...
};
```

往一个list尾部里添加一个元素函数是 
list *listAddNodeTail(list *list, void *value)
可以看到 value的类型是void *

而在redisServer.Clients链表中加入redisClient时，传入的value类型是redisClient *
```javascript

/*
 * 创建一个新客户端
 */
redisClient *createClient(int fd)
{
    // 分配空间
    redisClient *c = zmalloc(sizeof(redisClient));
    ...
    // 如果不是伪客户端，那么添加到服务器的客户端链表中
    if (fd != -1)
        listAddNodeTail(server.clients, c);
    ...    
    // 返回客户端
    return c;
}

list *listAddNodeTail(list *list, void *value)
{
    listNode *node;
    // 为新节点分配内存
    if ((node = zmalloc(sizeof(*node))) == NULL)
        return NULL;
    // 保存值指针
    node->value = value;
    // 目标链表为空
    if (list->len == 0)
    {
        list->head = list->tail = node;
        node->prev = node->next = NULL;
        // 目标链表非空
    }
    else
    {
        node->prev = list->tail;
        node->next = NULL;
        list->tail->next = node;
        list->tail = node;
    }
    // 更新链表节点数
    list->len++;
    return list;
}
```
从redisServer.Clients里取出来的value，也是当成redisClient *用的
```javascript
// 返回给定链表的表头节点
#define listFirst(l) ((l)->head)
// 返回给定节点的值
#define listNodeValue(n) ((n)->value)
int clientsCronHandleTimeout(redisClient *c){
	...
}
void clientsCron(void)
{
	   ...
	   while(...){
	   ...
        head = listFirst(server.clients);
        c = listNodeValue(head);
        if (clientsCronHandleTimeout(c))
            continue;
        ...
       }
}
```
将慢日志加入 redisServer.slowlog时，传入的值的类型是slowlogEntry *
```javascript
void slowlogPushEntryIfNeeded(robj **argv, int argc, long long duration) {
    ...
    // 如果执行时间超过服务器设置的上限，那么将命令添加到慢查询日志
    if (duration >= server.slowlog_log_slower_than)
        // 新日志添加到链表表头
        listAddNodeHead(server.slowlog,slowlogCreateEntry(argv,argc,duration));
	...
}
slowlogEntry *slowlogCreateEntry(robj **argv, int argc, long long duration){
	...
}
```
## Q2: list结构里的函数指针是怎么回事
虽然说list 可以不关心放进去的 value是什么数据类型，
但是提供了下面三个函数定义来帮助处理各种不同的类型的 value

 1. 节点值复制函数 `dup`, 
2.  节点值释放函数 `free`, 
3.  节点值对比函数 节点值对比函数 `match`

### 帮忙复制 list->dup函数
在复制链表时，如果dup不为空，则会调用dup对value进行复制操作
否则仅复制value这个指针
```javascript
/*
 * 复制整个链表。
 * 复制成功返回输入链表的副本，
 * 如果因为内存不足而造成复制失败，返回 NULL 。
 * 如果链表有设置值复制函数 dup ，那么对值的复制将使用复制函数进行，
 * 否则，新节点将和旧节点共享同一个指针。
 * 无论复制是成功还是失败，输入节点都不会修改。
 */
list *listDup(list *orig)
{
    ...
    // 设置节点值处理函数
    copy->dup = orig->dup;
    copy->free = orig->free;
    copy->match = orig->match;
    ...
    while ((node = listNext(iter)) != NULL)
    {
        void *value;
        // 复制节点值到新节点
        if (copy->dup)
        {
            value = copy->dup(node->value);
            if (value == NULL) {
                ...
                return NULL;
            }
        }
        else
            value = node->value;
        ...
    }
    ...
    return copy;
}
```
### 帮忙释放 list->free函数
在删除list中一个元素或者释放整个list时，
如果free被赋值，则会调用free对value进行释放

```javascript
/*
 * 释放整个链表，以及链表中所有节点
 */
void listRelease(list *list)
{
	...
    while (...)
    {
        next = current->next;
        // 如果有设置值释放函数，那么调用它
        if (list->free)
            list->free(current->value);
        // 释放节点结构
        zfree(current);
        current = next;
    }
    // 释放链表结构
    zfree(list);
}

/*
 * 从链表 list 中删除给定节点 node 
 * 对节点私有值(private value of the node)的释放工作由调用者进行。
 */
void listDelNode(list *list, listNode *node)
{
    ...
    // 释放值
    if (list->free)
        list->free(node->value);
    // 释放节点
    zfree(node);
    // 链表数减一
    list->len--;
}
```
### 帮忙找人 list->match函数
遍历list时，如果match字段不为空
将通过match指向的函数对比value与key是否匹配
否则直接判断key与value是否相等
```javascript
listNode *listSearchKey(list *list, void *key)
{
    listIter *iter;
    listNode *node;
    while ((node = listNext(iter)) != NULL)
    {
        // 对比
        if (list->match)
        {
            if (list->match(node->value, key))
            {
                listReleaseIterator(iter);
                // 找到
                return node;
            }
        }
        else
        {
            if (key == node->value)
            {
                listReleaseIterator(iter);
                // 找到
                return node;
            }
        }
    }
    listReleaseIterator(iter);
    // 未找到
    return NULL;
}
```

好了，这就是redis的list，一个双向链表。
list相关结构与函数定义放在了 adlist.h文件，实现则在adlist.c文件
贴一下 adlist.h
```javascript
#ifndef __ADLIST_H__
#define __ADLIST_H__
/*
 * 双端链表节点
 */
typedef struct listNode {
    // 前置节点
    struct listNode *prev;
    // 后置节点
    struct listNode *next;
    // 节点的值
    void *value;
} listNode;
/*
 * 双端链表迭代器
 */
typedef struct listIter {
    // 当前迭代到的节点
    listNode *next;
    // 迭代的方向
    int direction;
} listIter;
/*
 * 双端链表结构
 */
typedef struct list {
    // 表头节点
    listNode *head;
    // 表尾节点
    listNode *tail;
    // 节点值复制函数
    void *(*dup)(void *ptr);
    // 节点值释放函数
    void (*free)(void *ptr);
    // 节点值对比函数
    int (*match)(void *ptr, void *key);
    // 链表所包含的节点数量
    unsigned long len;

} list;
/* Functions implemented as macros */
// 返回给定链表所包含的节点数量
// T = O(1)
#define listLength(l) ((l)->len)
// 返回给定链表的表头节点
// T = O(1)
#define listFirst(l) ((l)->head)
// 返回给定链表的表尾节点
// T = O(1)
#define listLast(l) ((l)->tail)
// 返回给定节点的前置节点
// T = O(1)
#define listPrevNode(n) ((n)->prev)
// 返回给定节点的后置节点
// T = O(1)
#define listNextNode(n) ((n)->next)
// 返回给定节点的值
// T = O(1)
#define listNodeValue(n) ((n)->value)
// 将链表 l 的值复制函数设置为 m
// T = O(1)
#define listSetDupMethod(l,m) ((l)->dup = (m))
// 将链表 l 的值释放函数设置为 m
// T = O(1)
#define listSetFreeMethod(l,m) ((l)->free = (m))
// 将链表的对比函数设置为 m
// T = O(1)
#define listSetMatchMethod(l,m) ((l)->match = (m))
// 返回给定链表的值复制函数
// T = O(1)
#define listGetDupMethod(l) ((l)->dup)
// 返回给定链表的值释放函数
// T = O(1)
#define listGetFree(l) ((l)->free)
// 返回给定链表的值对比函数
// T = O(1)
#define listGetMatchMethod(l) ((l)->match)
/* Prototypes */
list *listCreate(void);
void listRelease(list *list);
list *listAddNodeHead(list *list, void *value);
list *listAddNodeTail(list *list, void *value);
list *listInsertNode(list *list, listNode *old_node, void *value, int after);
void listDelNode(list *list, listNode *node);
listIter *listGetIterator(list *list, int direction);
listNode *listNext(listIter *iter);
void listReleaseIterator(listIter *iter);
list *listDup(list *orig);
listNode *listSearchKey(list *list, void *key);
listNode *listIndex(list *list, long index);
void listRewind(list *list, listIter *li);
void listRewindTail(list *list, listIter *li);
void listRotate(list *list);
/* Directions for iterators 
 *
 * 迭代器进行迭代的方向
 */
// 从表头向表尾进行迭代
#define AL_START_HEAD 0
// 从表尾到表头进行迭代
#define AL_START_TAIL 1
#endif /* __ADLIST_H__ */

```
## 小结
又到了小结时间，跑个题。
复制函数dup，和匹配函数match 如果忘了设置，在调试阶段，比较容易发现。
list的free函数，如果value指向的结构需要释放，但是忘了设置，写完调试阶段相对较难发现，容易会出现内存泄漏。


 # 往期博客回顾
1. [redis服务器的部分启动过程](https://blog.csdn.net/a158372582/article/details/106023071)
2.  [GET命令背后的源码逻辑](https://editor.csdn.net/md/?articleId=106035658)
3. [redis的基础数据结构之 sds](https://blog.csdn.net/a158372582/article/details/106063645)

