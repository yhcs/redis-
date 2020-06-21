
@[TOC](redis持久化 之 反面面试官)
# 给新观众老爷的开场
大家好，我是弟弟！
最近读了一遍 黄健宏大佬的 **<<Redis 设计与实现>>**，对Redis 3.0版本有了一些认识
该书作者有一版添加了注释的 redis 3.0源码
[👉官方redis的github传送门](https://github.com/antirez/redis)。
[👉黄健宏大佬添加了注释的 redis 3.0源码传送门](https://github.com/huangz1990/redis-3.0-annotated.git)


网上说Redis代码写得很好，为了加深印象和学习redis大佬的代码写作艺术，了解工作中使用的redis 命令背后的源码逻辑，便有了写博客记录学习redis源码过程的想法。


# redis 持久化
## 面试官: "你了解redis的持久化吗？"
候选人: "
1. redis有两种持久化机制，一种是rdb，一种是aof
2. rdb可以理解为"全量备份",将当前全量的k/v键值对全部写到文件中。
3. aof可以理解为"增量备份",每当收到 部分相关的redis命令，将命令以文本字符串的形式追加写入文件中。
4. 如果启用rdb机制，在redis挂掉的时候，会丢失上一次rdb备份到现在之间的数据
5. 如果启用aof机制，在redis挂掉的时候，使用aof重放命令不如一份完整的rdb文件速度来得快。
6. 于是在redis4.0及以后aof重写时，可配置 aof与rdb混合的方式。
aof重写时，先保存当前全量rdb文件数据，保存期间新的命令以aof的形式追加写入，这样redis挂掉重启时，大部分数据可以快速恢复，之后的增量数据由aof记录，rdb与aof优势互补，真正的又快又稳😂  “

面试官: (嗯，这小子回答得还不错) 
## RDB
### 面试官: "你能说说rdb具体是怎么备份数据的吗？"
候选人: "
1. 备份rdb文件有两种方式
2.  一种是命令主动触发 save, bgsave
   2.1  save将阻塞redis主线程，
   2.2 bgsave 使用 fork+copy on write 开子进程来备份数据
   不会阻塞主线程，使用的内存也不会突然翻倍
3. 一种是通过配置文件配置自动备份rdb条件 比如 60秒内 >= 10000个k/v 有变化，5分钟内 >= 10个k/v有变化 将触发bgsave"

### 面试官: "那具体rdb的文件是如何生成的呢？"
候选人:（不好意思，我刚好看过一点）
写入rdb文件中的数据包括
1. 当前redis的所有k/v键值对，以及过期key的过期时间
2. 上述k/v分别属于哪一个db，也就是dbid，(因为恢复数据的时候需要使用）
3. 当前rdb文件格式的版本号

至于生成rdb文件的代码逻辑，拿bgsave命令来举例吧。

1. 判断 server.rdb_child_pid 是否等于 -1. 
1.1 等于-1表示当前没有子进程进行bgsave操作，本次bgsave可以正常执行
1.2 不等于-1，表示正在进行bgsave的子进程的pid，本次bgsave不会真正执行。
2. childpid = fork() 创建子进程
2.1 父进程会记录fork耗时,子进程pid,关闭自动rehash,标记 dict_can_resize = 0
2.2 子进程将关闭网络连接套接字，然后开始执行真正的rdbSave函数,在该函数中会生成一个完整的rdb文件

rdbSave函数将按照rdb文件格式写入数据，逻辑如下
1. 创建一个临时文件，用于写入数据
    ```javascript
    snprintf(tmpfile,256,"temp-%d.rdb", (int) getpid());
    fp = fopen(tmpfile,"w");
   ```
4. 写入 rdb版本号
    ```javascript
	 //The current RDB version. When the format changes in a way that is no longer
     //backward compatible this number gets incremented.
     //RDB 的版本，当新版本不向旧版本兼容时，增一
     #define REDIS_RDB_VERSION 6
    ... 
    snprintf(magic,sizeof(magic),"REDIS%04d",REDIS_RDB_VERSION);
    if (rdbWriteRaw(&rdb,magic,9) == -1) goto werr;
    ...
   ```
5. 跳过空的db字典
  写入db选择码 254 以及dbid， 记录接下来的数据，原来属于哪个db
6. 遍历当前db字典，取出key,value,以及key的过期时间
如果key过期了，那么这个k/v不会写入rdb文件
如果key没过期，写入过期时间，key，value 相关的值
如果key没有过期时间，仅写入 key, value 相关的值
7. 所有k/v写入之后，最后写入EOF码 255
8. 在写入 8个字节的 crc校验码
9. fflush、fsync、fclose 三连同步到文件并关闭fd
10. 子进程以一个退出码结束退出，表示 bgsave 执行 成功 or 失败
11. 主进程在每次 serverCron 时，会使用wait3()来获取结束子进程的信息
若获取到rdb子进程的结束信息 ，会将 server.rdb_child_pid 置为-1，并更新相关的信息。

面试官: "嗯，不错，回答得很详细"
候选人: "嗯，这个还没说完"
### 面试官: （难道这小子要反面我 😂）

候选人: "
对于具体的 key/value/过期时间 如何保存还没说，实际上这也是 保存rdb文件中骚操作最多的地方，应该需要重点讲一下。

> 实际上对于读/写文件来说，写入的内容生前到底是什么数据结构是不关心的。
读/写文件只关心，要从哪个地址开始，要操作多少个字节，完事。
什么跳跃表，哈希表， 站在读/写文件的角度来看实际上都一样。
当我看到这里的时候，才第一次体会到了什么叫  `文件流`。
1. 要在文件中保存 各种数据结构变量 是比较麻烦的，因为恢复的时候还要对从文件流里读出来的数据进行识别，哪一坨是什么结构。
为了简化问题，redis写入rdb文件的变量类型， 大概有三类

  2. `直接写入的 1字节 标识码`，比如 选择db的操作码 254, 结束标记EOF 255 
  3. `整数变量`，因变量大小不同，为了节省空间一共分为了 6位/14位/32位 三种数字，并且在低字节用额外2位 表示 类型码 也就是表示数字类型，这样恢复的时候才能读出正确的数字
  4. `字符串变量`
     4.1 一类是可以表示为数字的字符串变量，使用的表示方法跟整数变量类似。
     4.2 一类是不能表示为数字且不能被压缩的字符串变量，会将 字符串长度 以整形变量的形式保存下来，然后再写入字符串。
     4.3 一类是可以被压缩的字符串变量，会写入被压缩标识码，压缩后字符串长度，压缩钱字符串长度，压缩后字符串。

 
 有了上述的三类基本结构，拿 保存相对复杂一些的 zset 数据类型的操作，来讲讲一讲redis的键值对，最后是怎么保存在rdb中的
1.  如果key有过期时间，写入 过期时间标识码 252，且后面紧跟一个8字节的过期时间戳
3. 写入1个value的类型标识码 (实际上这个标识码 同时表示了 value的数据类型，编码类型 以及表示后面跟着的是一对 key/value ，因为key都是字符串类型，所以只记value的就好了)
   ```javascript
	/* Dup object types to RDB object types. Only reason is readability (are we
	 * dealing with RDB types or with in-memory object types?).
	 *
	 * 对象类型在 RDB 文件中的类型
	 */
	#define REDIS_RDB_TYPE_STRING 0
	#define REDIS_RDB_TYPE_LIST   1 //默认的双向链表实现的链表
	#define REDIS_RDB_TYPE_SET    2 //默认的哈希表实现的集合
	#define REDIS_RDB_TYPE_ZSET   3 //默认的跳跃表实现的有序集合
	#define REDIS_RDB_TYPE_HASH   4 //默认的哈希表实现的哈希数据类型
	
	/* Object types for encoded objects.
	 *
	 * 对象的编码方式
	 */
	#define REDIS_RDB_TYPE_HASH_ZIPMAP    9 
	#define REDIS_RDB_TYPE_LIST_ZIPLIST  10 //压缩列表实现的链表
	#define REDIS_RDB_TYPE_SET_INTSET    11 //整数集合实现的集合
	#define REDIS_RDB_TYPE_ZSET_ZIPLIST  12 //压缩列表实现的有序集合
	#define REDIS_RDB_TYPE_HASH_ZIPLIST  13 //压缩列表实现的哈希表
   ```
  4. 写入key对应的字符串
  5. 根据value的类型，处理有序集合(这里拿有序集合来举例子)
4.1 如果是压缩列表实现的，直接将 压缩列表占用的空间字节数作为字符串的长度，整个压缩列表 当作一个字符串，写入rdb文件。
      > 因为压缩列表本身就是紧凑的一串内存空间
  
     4.2 如果是跳跃表实现的，写入元素个数， 遍历跳跃表中的字典，将所有的 value/score 
     分别以 value 字符串形式， scroe 特殊标识码/或者字符串形式 写入rdb文件中
     > 因为 score是浮点数 正无穷，负无穷，或者不是一个数 可以用特殊标识符来表示，如果是正儿八经的浮点数，将转换成字符串来表示
     
     > `为什么这里只是遍历了 跳跃表中的字典取出所有 value/score 写入rdb呢?那这个跳跃表的层高呢，各个value的顺序，不存了吗？`
     是的，不存了。这些信息，其实在恢复rdb文件的时候 根据value/score是可以重建这个跳跃表的，而且减少这些冗余的信息，可以减少rdb文件的体积，提高rdb文件生成的速度。

"
候选人: "
1. 除了上述的这些为了节省rdb文件空间的骚操作以外，对字符串在一定情况下也是会进行压缩的，redis中使用了 lzf 压缩算法。可以通过配置启用该算法。"
2. 该算法具有 o(1xn)的时间复杂度，比较适合redis这种对性能有点要求的软件使用。 但快和高压缩率 这两者比较难兼得，该算法的压缩率是不稳定的。
不能保证一定能压缩，该算法无法对一些字符串进行压缩的情况也是有的。

   具体的原因在于 
2.1 该算法会从头到尾遍历需要压缩的字符串，
2.2 并采样统计 "第一次出现的子串"  
2.3 如果后面该子串重复出现，
2.4 记录该子串距离前面被重复的子串的偏移量，以及重复的长度就ok了 
2.5 显然该算法在字符串中存在大量重复子串的情况下，压缩率高
2.6 该算法在字符串中不存在大量重复子串的情况下，压缩率低
    >  对比测试了 一个长度为100万的字符串，在不同情况下的压缩率
    > 如果每个字符是在26个不同的字母中随机产生的 压缩后大小为原串的 91.5%
    > 如果每个字符是在10个不同的字母中随机产生的 压缩后大小为原串的 64.3%
    > 如果是100万个相同的字母，压缩后大小为原串的 1.13%
    > lzf算法，压缩/解压代码带注释不超过500行，可以说是很轻量了
    > [lzf压缩算法网站](http://liblzf.plan9.de/)
    > [lzf压缩算法源码下载](http://dist.schmorp.de/liblzf/)

"
面试官: (嗯，不错不错，这小子有点东西)
### 面试官: "redis是如何通过rdb文件恢复数据的呢？"
候选人: "
1. redis作为缓存服务器启动时，如果开启了rdb，会根据配置查找的rdb文件
2. 找到了的话，其实按照写入rdb文件的顺序和格式解出数据就行了。
3. 但是仅靠rdb中读出来的数据还不能完整的还原对应的数据结构，
3. 比如上面所说的zset，当发现要读一个zset时，
redis会像平时创建zset一样，创建一个zset数据结构，
并将各个value/score 模拟添加进去，以此还原出正确的数据结构
"

## AOF
### 面试官: "AOF是怎么备份数据的？"
候选人: "
如果 配置文件中 appendonly 设置为yes 启动redis的话，AOF功能将被开启。
1. 当redis处理一个命令后，计算dirty值(该命令有没有修改内存数据)
2. 如果该命令修改了内存数据，用该命令 [RESP协议格式](http://www.redis.cn/topics/protocol.html)的字符串先写入全局的AOF缓冲区，也就是server.aof_buf。
如果有必要指明命令是写到哪个db的话，还会写入一个选择db号的命令
    > 比如 "SET mykey myvalue" 命令， 实际写入的字符串是 "*3\r\n$3\r\nSET\r\n$5\r\nmykey\r\n$7\r\nmyvalue\r\n"
    实际上redis客户端发到redis服务器的命令就是长这个样子的。
    以原命令字符串的形式记录AOF文件，在使用AOF文件恢复数据时比较方便。在内部起一个伪客户端，直接重放命令恢复数据
 3. 在redis主线程的循环逻辑里，每轮事件轮训前都会执行 beforeSleep 函数，该函数中会将 server.aof_buf的内容尝试写入aof文件并同步
    3.1 如果后台线程还有同步到文件的任务没完成，就会等一会儿再写
    3.2 如bgsave,bgrewrite没执行完，也等一会儿再同步
 5. 写入aof_buf到文件之后，就是AOF的同步策略了，同步策略是可配置的。
    ```javascript
	/* Append only defines */
	#define AOF_FSYNC_NO 0          //不主动触发同步，依赖系统自己同步
	#define AOF_FSYNC_ALWAYS 1      //总是同步
	#define AOF_FSYNC_EVERYSEC 2    //超过1秒同步一次
	#define REDIS_DEFAULT_AOF_FSYNC AOF_FSYNC_EVERYSEC //默认同步策略
    ```
 
      4.1    对于 AOF_FSYNC_ALWAYS 策略，每次将aof_buf写入aof文件后，就在当前的主线程中调用fsync函数同步。
      4.2   对于 AOF_FSYNC_EVERYSEC 策略，若超过1秒后，会将本次同步任务，使用锁的方式，写入全局任务队列。在别的线程中取出该任务调用 fsync函数同步到文件。
      >  没错，这里会用到锁。redis内部，对于文件同步是有另外的线程来单独处理的，所以这里会有一点点 使用锁来解决多线程的共享变量读写问题。

### 面试官: "AOF文件有什么可以优化的地方吗？"
候选人: (这不就是 AOF文件重写吗，还好我看过 😂)
"
1. 像上述的AOF文件，会存下来一些冗余的命令。
   >比如设置了一个过期key/value，一定时间之后，这个key/value实际已经不需要了。
又比如设置了一个key/value，后面这个key/value 被删掉了，那AOF文件里，该key/value的设置/删除命令，实际上就是冗余的。

2. 这个时候就需要AOF文件重写了，将原本文件里的冗余命令全部删掉，只保留恢复数据需要的最少的信息就可以了，这样AOF的文件大小会缩小，包含的命令数也会减少，自然使用AOF文件恢复数据是，也会更快一些。

3. 当然AOF重写命令，上面也说了。
在4.0之后 AOF重写是可以先保存一份完整的rdb文件，然后再rdb文件之后再追加写入增量的AOF文件内容。

4. 在4.0之前AOF重写实际上跟rdb文件逻辑类似，会遍历各个db，并且将现存的key/value/过期时间这些信息取出来，然后转换成 “设置命令” 存入新的AOF文件中，老的AOF文件就可以不再使用了，新AOF文件具有恢复数据需要的最少命令。
"
### 面试官: "AOF重写的逻辑你了解吗？"
候选人:  "
1. AOF重写逻辑跟RDB其实有点类似，也是fork+copy on write开子进程来完成AOF文件的重写，并将子进程的pid记录到 全局变量 server.aof_child_pid 中。

2. 与RDB不太一样的地方
一个是上面说的，AOF会将当前DB数据以 "设置命令"的方式写入AOF文件。

3. 另外一个就是，AOF文件写入期间，发生的新的需要写入AOF文件的命令，会被记录到全局的server.aof_rewrite_buf_blocks 中，该变量实际上是一个list，每一个list节点会有一块儿buffer来存放增量命令，默认每块是10MB。
4. 当AOF重写的子进程结束后返回一个退出码，主进程在serverCron的轮询中调用 wait3()来接收AOF重写子进程的结束退出信息。收到后，会将 server.aof_rewrite_buf_blocks中的内容在主线程中写入AOF文件。并将 server.aof_rewrite_buf_blocks, server.aof_buf清空，将server.aof_child_pid 置为-1
    > 将 server.aof_rewrite_buf_blocks 写入 AOF 文件 这个操作会阻塞主进程，但如果不这么操作再开子进程来写，这会不会陷入一个久久不能结束的循环 🤔

5. 4.0之后如果开了AOF-RDB混合持久化，AOF重写时全量数据保存那里换成了保存rdb文件，另外恢复的时候也是用RDB恢复，再用AOF增量恢复，其余逻辑倒是差不太多。
"
## 面试官: "最后一个问题" 
面试官: "你明天能来上班吗？"
候选人: "😏"


# 小结
1. 从全局来看，redis的数据备份实际上就是 全量 + 增量 两种方式。
2. 但毕竟这是程序，在实际实现的时候，涉及到的细节数量还是比较多的。
3. redis的数据备份与恢复 一方面可以用来给自己这个进程用，另一方面主从复制的似乎也是有点用的，这个等写主从复制的时候再看看。


 # 往期博客回顾
1. [redis服务器的部分启动过程](https://blog.csdn.net/a158372582/article/details/106023071)
5.  [GET命令背后的源码逻辑](https://editor.csdn.net/md/?articleId=106035658)
6. [redis的基础数据结构之 sds](https://blog.csdn.net/a158372582/article/details/106063645)
7. [redis的基础数据结构之 list](https://blog.csdn.net/a158372582/article/details/106086284)
8. [redis的基础数据结构 之 ziplist](https://blog.csdn.net/a158372582/article/details/106107759)
9. [redis 基础数据结构之 hash表](https://blog.csdn.net/a158372582/article/details/106234075)
10. [redis不稳定字典的遍历](https://blog.csdn.net/a158372582/article/details/106304649)
11. [redis 基础数据结构 之 集合](https://blog.csdn.net/a158372582/article/details/106202553)
12. [redis 基础数据结构 之 有序集合](https://blog.csdn.net/a158372582/article/details/106630445) 
13. [redisObject 以及 对抽象的理解](https://blog.csdn.net/a158372582/article/details/106712261)