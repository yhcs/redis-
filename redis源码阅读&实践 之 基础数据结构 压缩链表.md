
@[TOC](redis源码阅读&实践 之 基础数据结构压缩链表)
# 给新观众老爷的开场
大家好，我是弟弟！
最近读了一遍 黄健宏大佬的 **<<Redis 设计与实现>>**，对Redis 3.0版本有了一些认识，该书作者有一版添加了注释的 redis 3.0源码
[👉官方redis的github传送门](https://github.com/antirez/redis)。
[👉黄健宏大佬添加了注释的 redis 3.0源码传送门](https://github.com/huangz1990/redis-3.0-annotated.git)


网上说Redis代码写得很好，为了加深印象和学习redis大佬的代码写作艺术，了解工作中使用的redis 命令背后的源码逻辑，便有了从redis命令角度学习redis源码的想法。
(全文提到的redis服务器，都指在 **mac os 上启动的一个默认配置的单机redis服务器**)

# ziplist是什么？
redis的链表数据类型，在底层有两种数据结构实现.
一种是[上一篇讲到list数据结构](https://blog.csdn.net/a158372582/article/details/106086284),
另外一种就是 `压缩链表 ziplist`。
那这个ziplist又是什么呢？

弟弟:"从名字上看，压缩列表，就是一个被压缩了的列表"
观众老爷: "...，要你有何用，你不说我们也能看出来。😡"
弟弟:"...不好意思，打扰了🙃️"
弟弟: "`redis的压缩列表,存储在连续的内存空间中,没有代码层面的数据结构定义,依赖设计好的二进制格式来进行操作。`"
观众老爷:"你这讲得不够专业啊，也不形象。"
弟弟:"😭，好的了解。
那让我们来 结合 redis里的英文注释 和 黄健宏大佬的中文注释 来看看是怎么讲的"
（ps: 该注释出现在 ziplist.c 文件中）

## ziplist的设计目的
👇
```javascript
 * The ziplist is a specially encoded dually linked list that is designed
 * to be very memory efficient. 
 * Ziplist 是为了尽可能地节约内存而设计的特殊编码双端链表。
```
## ziplist的作用
👇
```javascript
 * It stores both strings and integer values,
 * where integers are encoded as actual integers instead of a series of
 * characters. 
 * Ziplist 可以储存字符串值和整数值，
 * 其中，整数值被保存为实际的整数，而不是字符数组。
```
## ziplist的小瑕疵
👇
```javascript
 * It allows push and pop operations on either side of the list
 * in O(1) time. However, because every operation requires a reallocation of
 * the memory used by the ziplist, the actual complexity is related to the
 * amount of memory used by the ziplist.
 * Ziplist 允许在列表的两端进行 O(1) 复杂度的 push 和 pop 操作。
 * 但是，因为这些操作都需要对整个 ziplist 进行内存重分配，
 * 所以实际的复杂度和 ziplist 占用的内存大小有关。
```
## ziplist 的空间布局
ziplist的空间布局，按低地址向高地址的方向 如下
1. 开头用 `四个字节来存放ziplist的字节数大小`
2. 接着用 `四个字节来存放ziplist尾节点的偏移字节数`
3. 然后是 `2个字节表示该ziplist里放了多少个节点`
4. 接着就是 `<节点>,<节点>,<节点>... 0到n个`
5. 最后是 `一个字节的ziplist结束标志 255`

👇看一遍注释
```javascript
 * ZIPLIST OVERALL LAYOUT:
 * Ziplist 的整体布局：
 * The general layout of the ziplist is as follows:
 * 以下是 ziplist 的一般布局：
 * <zlbytes><zltail><zllen><entry><entry><zlend>
```
### zlbytes的含义
```javascript
 * <zlbytes> is an unsigned integer to hold the number of bytes that the
 * ziplist occupies. This value needs to be stored to be able to resize the
 * entire structure without the need to traverse it first.
 * <zlbytes> 是一个无符号整数，保存着 ziplist 使用的内存数量。
 * 通过这个值，程序可以直接对 ziplist 的内存大小进行调整，
 * 而无须为了计算 ziplist 的内存大小而遍历整个列表。
```
### zltail的含义
```javascript
 * <zltail> is the offset to the last entry in the list. This allows a pop
 * operation on the far side of the list without the need for full traversal.
 * <zltail> 保存着到达列表中最后一个节点的偏移量。
 * 这个偏移量使得对表尾的 pop 操作可以在无须遍历整个列表的情况下进行。
```
### zllen的含义
```javascript
 * <zllen> is the number of entries.When this value is larger than 2**16-2,
 * we need to traverse the entire list to know how many items it holds.
 * <zllen> 保存着列表中的节点数量。
 * 当 zllen 保存的值大于 2**16-2 时，（大于等于 65535 时）
 * 程序需要遍历整个列表才能知道列表实际包含了多少个节点。
```
 ### zlend的含义
 ```javascript
 * <zlend> is a single byte special value, equal to 255, which indicates the
 * end of the list.
 * <zlend> 的长度为 1 字节，值为 255 ，标识列表的末尾。
 ```
观众老爷:"wo*, 这个ziplist的格式有点复杂啊"
弟弟:"是啊，有点复杂，还有ziplist的节点格式没说呢 ，
ziplist的节点格式 及 相关代码 是造成ziplist复杂的真正元凶🙃️"

### ziplist的节点格式
`ziplist节点` 的空间布局，大致分为三部分。
1. 描述前一个节点占用的空间大小
2. 描述本节点的值的类型，长度
3. 值本身

详细结构，按低地址向高地址的方向 如下
1. `1or5个字节`表示`前面一个节点的 字节数大小`。 
若第1个字节 小于 `#define ZIP_BIGLEN 254`，则该字节直接表示前一个节点的字节数大小
否则为5个字节，第2,3,4,5字节一起构成一个 32为整数表示前一节点的大小
2. 紧接着的 `1or2or5 字节`一共表示了三类信息
1-  `当前节点值类型`，
2- `描述当前节点值的长度所需要的空间大小的类型`，
3- `当前节点值的长度` ....
这也太能省了，通过提高结构的复杂程度弄晕计算机来节省存储空间 🙃️

	2.1 第一个字节叫 `encoding`
	如果 encoding 小于 `#define ZIP_STR_MASK 0xc0`(二进制 11000000),
	那该节点存的是字符串类型。
	那么 描述该字符串长度所需要的存储空间大小的类型就是 encoding & ZIP_STR_MASK
	取值分别以下三种,
	```javascript
	`/** 字符串编码类型
                                            //0x3f  00111111
                                            //0xc0  11000000
    #define ZIP_STR_06B (0 << 6)            //      00000000
   #define ZIP_STR_14B (1 << 6)            //      01000000
   #define ZIP_STR_32B (2 << 6)            //      10000000
   `
	```
	
	`6B,14B,32B的意思是,描述字符串长度所需要的空间分别是6位/14位/32位，`
	`且描述字符串的长度，是按大端序存放的，且前6位存放在了encoding里,额外的空间紧接着encoding`
	`所以描述这三种 字符串类型、存储字符串长度的空间大小，以及字符串的长度 一共需要 1/2/5个字节`
	
	如果一下就看懂了，那说明你真是个聪明的观众老爷呢😊
	🙃️如果看晕了，没关系，我们对着整数类型再来一遍😊	
	
	如果 encoding 大于等于 `#define ZIP_STR_MASK 0xc0`(二进制 11000000)
	那该节点的值是一个整数
	那么 描述该字符串长度所需要的存储空间大小的类型 就是 encoding本身 👇
	```javascript
	/* 
	* 整数编码类型
	*  */
	* #define ZIP_INT_16B (0xc0 | 0 << 4)     //      11000000
	* #define ZIP_INT_32B (0xc0 | 1 << 4)     //      11010000
	* #define ZIP_INT_64B (0xc0 | 2 << 4)     //      11100000
	* #define ZIP_INT_24B (0xc0 | 3 << 4)     //      11110000
	* #define ZIP_INT_8B 0xfe                 // 0xFE 11111110
	```
	
	8B/16B/24B/32B/64B 对应的 值的长度 分别是 (8位/16位/24位/32位/64位)
	所以整数类型与字符串类型不同的地方在于
	`ziplist整数类型节点，仅需要一个字节，就可以描述存储的值的是数字类型，和值占用的空间字节数 `
	
	整数还有一个隐藏类型，如果是整数[0,12], 则将值直接保存到了encoding中😏
	encoding取值在下面两个值之间(包含),那么encoding - ZIP_INT_IMM_MIN - 1 的结果就直接是整数值,自然地该节点不需要再存储一份值。
	`#define ZIP_INT_IMM_MIN 0xf1  /* 11110001 */`
`#define ZIP_INT_IMM_MAX 0xfd  /* 11111101 */`
3. 第三部分就是节点的值本身啦，在3.0版本中值被本身是没有经过压缩处理的。

# ziplist insert!
按照国际惯例啊，redis是有 key/value 数据库的说法的。
既然ziplist这么复杂，那我们就简单一点，站在crud的角度撸它一下。
在创建ziplist 创建一个insert插入一下之前。
我们来看一些源码，回忆一下ziplist的结构

1. 查看一个ziplist的大小 (占用的字节数)
```javascript
// 定位到 ziplist 的 bytes 属性，该属性记录了整个 ziplist 所占用的内存字节数
// 用于取出 bytes 属性的现有值，或者为 bytes 属性赋予新值
#define ZIPLIST_BYTES(zl) (*((uint32_t *)(zl)))
```
2. 查看ziplist的尾节点的偏移量
 ```javascript
// 定位到 ziplist 的 offset 属性，该属性记录了到达表尾节点的偏移量
// 用于取出 offset 属性的现有值，或者为 offset 属性赋予新值
#define ZIPLIST_TAIL_OFFSET(zl) (*((uint32_t *)((zl) + sizeof(uint32_t))))
```

3. 查看ziplist的节点个数
 ```javascript
// 定位到 ziplist 的 length 属性，该属性记录了 ziplist 包含的节点数量
// 用于取出 length 属性的现有值，或者为 length 属性赋予新值
#define ZIPLIST_LENGTH(zl) (*((uint16_t *)((zl) + sizeof(uint32_t) * 2)))
```
4. 查看 ziplist 表头的大小
  ```javascript
// 返回 ziplist 表头的大小
#define ZIPLIST_HEADER_SIZE (sizeof(uint32_t) * 2 + sizeof(uint16_t))
```
5. 查看 ziplist 表头的大小
  ```javascript
// 返回指向 ziplist 第一个节点（的起始位置）的指针
#define ZIPLIST_ENTRY_HEAD(zl) ((zl) + ZIPLIST_HEADER_SIZE)
```
6. 查看 ziplist 最后一个节点
```javascript
// 返回指向 ziplist 最后一个节点（的起始位置）的指针
#define ZIPLIST_ENTRY_TAIL(zl) ((zl) + intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl)))
```
7. 查看 ziplist 最后一个节点
```javascript
// 返回指向 ziplist 末端 ZIP_END （的起始位置）的指针
#define ZIPLIST_ENTRY_END(zl) ((zl) + intrev32ifbe(ZIPLIST_BYTES(zl)) - 1)
```

8. 
## 小结
又到了小结时间，跑个题。
复制函数dup，和匹配函数match 如果忘了设置，在调试阶段，比较容易发现。
list的free函数，如果value指向的结构需要释放，但是忘了设置，写完调试阶段相对较难发现，容易会出现内存泄漏。


 # 往期博客回顾
1. [redis服务器的部分启动过程](https://blog.csdn.net/a158372582/article/details/106023071)
2.  [GET命令背后的源码逻辑](https://editor.csdn.net/md/?articleId=106035658)
3. [redis的基础数据结构之 sds](https://blog.csdn.net/a158372582/article/details/106063645)
4. [redis的基础数据结构之 list](https://blog.csdn.net/a158372582/article/details/106086284)
