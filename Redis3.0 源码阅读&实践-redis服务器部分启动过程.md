@[TOC](Redis3.0 源码阅读&实践)


大家好，我是弟弟！最近读了一遍 黄健宏大佬的 **<<Redis 设计与实现>>**，对Redis 3.0版本有了一些认识，该书作者有一个添加了注释的 redis 3.0源码。

网上说Redis代码写得很好，为了加深印象和学习redis大佬的代码写作艺术，了解工作中使用的redis 命令背后的源码逻辑，便有了从redis命令角度学习redis源码的想法。

## 从唯一会用的 GET/SET 命令开始 😃
在我们胸有成竹的准备打出GET命令前，我们需要启动一个redis服务器😭，用来接收和处理我们发送的命令 。
(全文提到的redis服务器，都指在 **mac os 上启动的一个默认配置的单机redis服务器**)
那我们先把redis服务器搞起来
 1. **在redis 3.0源码中加入各种printf打印调试信息**
 2. sudo make install
 3. redis-server **启动！**
![redis3.0 单机服务器启动截图](https://img-blog.csdnimg.cn/20200509133710481.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ExNTgzNzI1ODI=,size_16,color_FFFFFF,t_70)
### 关于redis是单线程的说法
好了，单机redis服务器搞起来了，不禁想起redis是单线程这么一个说法。那这根线程是啥呢，这根线程是一执行就直接returen了吗？显然不是，如果直接就return了，redis服务器启动之后就退出了，那谁来干redis服务器该干的事情😱
嗯，从启动截图来看，这根线程一直在循环，因为没有退出。真是简单直接的原因 😏

**那这根线程到底是啥呢？**
其实就是redis的main函数。
在 redis.c/main 中可以看到下面的代码，
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200509141242268.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ExNTgzNzI1ODI=,size_16,color_FFFFFF,t_70)
ae.c/aeMain 当stop为false时，会一直循环下去。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200509141505349.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ExNTgzNzI1ODI=,size_16,color_FFFFFF,t_70)
**那这根线程在干嘛**
1. 省略n多细节，来到 redis.c/main里的initServerConfig 初始化服务器相关配置， 比如初始化默认端口 6379
2. 省略n多细节，来到 redis.c/main里的initServer 创建并初始化服务器数据结构
3. ....省略1万字...
4. 开启事件处理循环
观众: "emmm... 你tm在逗我,这能说清楚个啥？"
弟弟: "好吧，让我们抛开源码拍一下脑袋，redis起一个线程一直循环，到底在搞什么。"
弟弟: "啊，我想到了。要接受客户端的链接啊！"
观众: "嗯，弟弟想法很不错，居然能想到要接受客户端的链接 😤"

### i/o多路复用
**没错这根线程干得其中一件事情就是 调用系统函数创建一个i/o对象并拿到文件描述符，读取/处理 客户端的网络连接请求，都跟这个文件描述符有关**
这个i/o对象 具体是 select,poll,epoll,kqueue 中的哪一个，取决于编译时的具体操作系统，就像上面说的这是在 mac os 上编译的 redis 3.0 服务器，

可以看到 当前os中 使用的是 kqueue，从这个if/else里可以体现出一个 引入的先后顺序![在这里插入图片描述](https://img-blog.csdnimg.cn/20200509143537661.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ExNTgzNzI1ODI=,size_16,color_FFFFFF,t_70)
在上述这4个 7，8年前的 文件 都实现了 aeApiCreate函数，而且不仅实现了这一个，这4个文件里分别实现了一套同名字的 事件处理函数。这个操作可以理解为对事件处理的抽象，在go语言里跟接口的概念很像，这里不禁要说一句**🐂🍺**。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200509143344106.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ExNTgzNzI1ODI=,size_16,color_FFFFFF,t_70)
再看 ae_kqueue.c/aeApiCreate 干了啥，调用系统函数 kqueue()，返回了一个kqfd文件描述符，并保存在全局的server结构里的事件相关的结构字段中，
这个kqfd就是实际 进行传说中i/o多路复用 的文件描述符...， 读取并处理 客户端的链接请求，都跟这个kqfd相关
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200509144641266.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ExNTgzNzI1ODI=,size_16,color_FFFFFF,t_70)
好了 一顿操作下来 kqueue 对象创建完了，kqfd文件描述符也拿到了，那我就要开始读取/处理 来自客户端的链接请求啦！ 
等等，等等...还没完

用kqfd 去哪儿 读取数据呢？好像有点问题。😰
select，poll这种 无差别遍历全局i/o事件的，似乎是可以直接开始遍历了。
但是网上不都说 epoll,kqueue 跟 select, poll 不一样吗，
**在活跃的网络连接较少的情况下，epoll，kqueue对活跃链接回调处理函数的方式，避免了无差别遍历较多不活跃没有事件消息的链接 来提高了i/o读取效率。**

是的，我们还没有将要监听 哪个文件描述符的哪种事件 告诉 kqfd。
先要有要监听的文件描述符，这个文件描述符 应该是 listen本地端口的tcp 的socket


### 监听默认端口6379
让我们来看看这根线程干得另外一个事情就是，调用系统网络相关函数(socket, bind, listen)，创建 listen 6379端口 的socket![在这里插入图片描述](https://img-blog.csdnimg.cn/20200509152007285.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ExNTgzNzI1ODI=,size_16,color_FFFFFF,t_70)
从源码里可以看到，分别创建了 ipv4、ipv6的socket 来listen 端口 6379, 并将对应的文件描述符保存了起来。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200509152044716.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ExNTgzNzI1ODI=,size_16,color_FFFFFF,t_70)


### 源码里具体的i/o多路复用
到这里，我们有了三个文件描述符，一个是 kqfd, 一个ipv4的socket的fd，一个ipv6的socket的fd。
让我们来看下源码，这一个for循环 就将 两个socket的fd 的可读事件及可读事件的 事件处理函数acceptTcpHandler 设置到了 kqfd上。
kqfd就能够监听 客户端对端口6379的连接请求了
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200509152525371.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ExNTgzNzI1ODI=,size_16,color_FFFFFF,t_70)
回头看下最开始的那个 while循环， 里面有个 aeProcessEvents
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200509153147840.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ExNTgzNzI1ODI=,size_16,color_FFFFFF,t_70)
该函数里使用 aeApiPoll 调用系统 kevent 函数收集 绑定在kqfd上 的活跃事件信息
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200509153622521.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ExNTgzNzI1ODI=,size_16,color_FFFFFF,t_70)
收集到活跃事件后，就开始调用之前绑定的事件处理函数
这里的 fe->rfileProc 就是读文件事件处理函数, fe->wfileProce则是写文件事件处理函数
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200509154014427.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ExNTgzNzI1ODI=,size_16,color_FFFFFF,t_70)
我们再来看下一下redis服务器的启动截图
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020050915495479.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ExNTgzNzI1ODI=,size_16,color_FFFFFF,t_70)
可以看到，kqfd = 3，( ipv4socketfd=5, ipv6socketfd=4, 且绑定的读事件处理函数都是acceptTcpHandler)
### 事件处理器循环 与 客户端链接
好吧，讲了这么多了，我是不是要开始GE...
不慌，我们还要先启动redis客户端连接到redis服务器 😭
 1. **redis-cli -h 127.0.0.1 -p 6379** 启动!

让我们先看一张图
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200509173151522.jpeg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ExNTgzNzI1ODI=,size_16,color_FFFFFF,t_70)
客户端小2 向服务器 1 一个连接请求，可以看到 1号服务器我们自己加的调试信息
ipv4socketfd=5 这个fd 触发了它的读事件处理函数acceptTcpHandler
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200509161908381.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ExNTgzNzI1ODI=,size_16,color_FFFFFF,t_70)
acceptTcpHandler 里调用 anetTcpAccept 来接收链接
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200509160323448.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ExNTgzNzI1ODI=,size_16,color_FFFFFF,t_70)
这里调用系统 accept函数 接收链接并返回一个 fd
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200509160404750.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ExNTgzNzI1ODI=,size_16,color_FFFFFF,t_70)
从日志上看到的操作是

  1. fd: 5, 收到来自 ip: 127.0.0.1, port: 52806 的链接请求 创建的socket对应的fd为: 6
  2. 对fd: 6, 设置读请求事件处理器 readQueryFromClient: c916100
  3. 设置 kqfd: 3, 对 fd: 6 读事件的监听

接收完一个客户端链接后，acceptTcpHandler 调用 acceptCommonHandler ，
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200509162042434.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ExNTgzNzI1ODI=,size_16,color_FFFFFF,t_70)
acceptCommonHandler 调用 createClient 创建一个redisClient对象并初始化，并且给新生成的 fd: 6 绑定读取时间处理函数readQueryFromClient， 这个函数才是用来处理客户端命令的 
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200509162329630.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ExNTgzNzI1ODI=,size_16,color_FFFFFF,t_70)


可以看出 最开始的 fd: 5, fd: 4 是用来 接受/处理 客户端链接用的
fd: 6 是服务器跟链接上来的客户端进行命令交互使用的fd。 且绑定的读取事件处理函数为 readQueryFromClient


**在服务器接受一个客户端链接之后，会创建一个 redisClient 对象。
不同的客户端有对应的 文件描述符，这是服务器给客户端回复消息用的，并且存在了redisClient->fd 里，而redisClient放在全局的 redisServer->clients 中.**

以下是 redisClient 与 redisServer  结构的局部字段
为避免引起不适，省略n多字段

```javascript
/* With multiplexing we need to take per-client state.
 * Clients are taken in a liked list.
 *
 * 因为 I/O 复用的缘故，需要为每个客户端维持一个状态。
 *
 * 多个客户端状态被服务器用链表连接起来。
 */
 //为避免引起不适，省略n多字段
typedef struct redisClient
{
    // 套接字描述符
    int fd;
    // 当前正在使用的数据库
    redisDb *db;
    // 当前正在使用的数据库的 id （号码）
    int dictid;
    // 客户端的名字
    robj *name; /* As set by CLIENT SETNAME */
    // 查询缓冲区
    sds querybuf;
    // 查询缓冲区长度峰值
    size_t querybuf_peak; /* Recent (100ms or more) peak of querybuf size */
    // 参数数量
    int argc;
    // 参数对象数组
    robj **argv;
    // 记录被客户端执行的命令
    struct redisCommand *cmd, *lastcmd;
    // 已发送字节，处理 short write 用
    int sentlen; /* Amount of bytes already sent in the current
                               buffer or object being sent. */
    // 创建客户端的时间
    time_t ctime; /* Client creation time */
    // 客户端最后一次和服务器互动的时间
    time_t lastinteraction; /* time of the last interaction, used for timeout */
    /* Response buffer */
    // 回复偏移量
    int bufpos;
    // 回复缓冲区
    char buf[REDIS_REPLY_CHUNK_BYTES];

} redisClient;

//为避免引起不适，省略n多字段
struct redisServer
{
    // 数据库
    redisDb *db;
    // 命令表（受到 rename 配置选项的作用）
    dict *commands; /* Command table */
    // 命令表（无 rename 配置选项的作用）
    dict *orig_commands; /* Command table before command renaming. */
    // 事件状态
    aeEventLoop *el;
    /* Networking */
    // TCP 监听端口
    int port; /* TCP listening port */
    // 描述符
    int ipfd[REDIS_BINDADDR_MAX]; /* TCP socket file descriptors */
    // 描述符数量
    int ipfd_count; /* Used slots in ipfd[] */
    // 一个链表，保存了所有客户端状态结构
    list *clients; /* List of active clients */

};
```

小结下
  1. **服务器一顿初始化，捋到的点是 3个fd， 分别是 kqfd, i/o对象的文件描述符，和另外两个 监听端口6379的套接字描述符，一个ipv4的一个ipv6的，并设置后两个fd的读事件请求处理器 acceptTcpHandler，并绑定到kqfd上。**
  2. **客户端链接服务器，服务器对应的fd会有活跃的读取事件，使用 i/o对应描述符读取到之后，使用对应fd绑定的读取事件处理函数进行处理，这个函数 acceptTcpHandler 是用来处理客户端链接请求的。**
  3. **acceptTcpHandler 会创建一个redisClient 并初始化结构， 会用到accept链接之后产生的fd，该fd绑定了一个读事件处理函数readQueryFromClient， 这个函数是用来处理客户端请求的命令的。**
  4.  **可以简单看出，redis服务器是事件驱动的，上述属于文件事件**
  
  好了，服务器搞起来了，客户端也连接上了，处理客户端的命令的函数也有了，我们来愉快的发送GET请求吧。

此时想起女朋友说她晚上想吃的那家涮肚。要早点去才吃得上。😱
好了GET请求就留到下篇文章再发。

观众:"说好了从GET/SET请求说起，结果搞了半天都还没开始说 ，你个臭弟弟😡"
  


  fd: 6, 调用读事件处理器 fe->rfileProc: c181020, 准备读入数据
fd: 6 readQueryFromClient 开始处理读事件
fd: 6, 命令信息->  参数个数: 2, 参数:  dbg mylist 
查找命令列表
dbg命令 开始执行 
mylist 链表的实现方式是 压缩链表
	元素个数为: 512
	结构占用大小为: 1035字节
fd: 6  安装写处理器 sendReplyToClient: c1812d0
设置 kqfd: 3, 对 fd: 6 写事件的监听
dbg命令 执行结束
fd: 6 readQueryFromClient 结束处理读事件

在第1589619582380123 微秒 本次事件循环结束

fd: 6, 调用写事件处理器  fe->wfileProc: c1812d0, 准备写入数据
fd: 6 sendReplyToClient 开始处理写入事件
+OK
以上为写入数据内容
删除 kqfd: 3, 对 fd: 6 写事件的监听
fd: 6 sendReplyToClient 结束处理写入事件



dbg命令 开始执行 
mylist 链表的实现方式是 普通链表, list结构大小: 48
	元素个数为: 512
	结构占用大小为: 12336字


  mylist 链表的实现方式是 压缩链表
	元素个数为: 512
	结构占用大小为: 1547字节
fd: 6  安装写处理器 sendReplyToClient: a0172d0


mylist 链表的实现方式是 普通链表, list结构大小: 48
	元素个数为: 512
	结构占用大小为: 12336字节
