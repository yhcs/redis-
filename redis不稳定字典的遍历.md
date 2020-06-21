
@[TOC](redis 不稳定字典的遍历)
# 给新观众老爷的开场
大家好，我是弟弟！
最近读了一遍 黄健宏大佬的 **<<Redis 设计与实现>>**，对Redis 3.0版本有了一些认识
该书作者有一版添加了注释的 redis 3.0源码
[👉官方redis的github传送门](https://github.com/antirez/redis)。
[👉黄健宏大佬添加了注释的 redis 3.0源码传送门](https://github.com/huangz1990/redis-3.0-annotated.git)
网上说Redis代码写得很好，为了加深印象和学习redis大佬的代码写作艺术，了解工作中使用的redis 命令背后的源码逻辑，便有了写博客记录学习redis源码过程的想法。


# redis 不稳定字典的遍历
众所周知 `HSCAN`命令可以对redis里的一个hash表的k/v进行遍历，
但每次只返回部分数据、和下一次遍历需要用到的游标。
该命令能保证指定字典可以遍历所有k/v不遗漏，但可能会有重复
[没用过hscan命令？](https://www.runoob.com/redis/hashes-hscan.html)

这篇文章对redis的不稳定字典遍历进行一个大致的描述...

先看下一结论，提前有个映像 🙃️
`将哈希数组下标与游标在逻辑上把其二进制的高低位反转，且计算下一次游标时根据大表掩码长度反转游标，从反转后的低位+1的技巧，利用 游标 单调递增 的性质。遍历时不遗漏 k/v ，扩容时不会重复遍历，缩容可能有部分重复遍历。`


阅读前建议先通过目录找到 dictScan源码与注释，
先看一遍源码，才知道下面的内容再说什么
写得可能不太好，容易绕晕😷。


## redis字典的状态
稳定字典的遍历在两次迭代遍历之间，字典不会有变化。容易遍历，也容易理解。
不稳定字典在两次迭代遍历间，字典会发生变化。

什么是稳定字典、不稳定字典？
- 稳定字典 说的是 当前没有进行rehash的字典。意思是要么没进行过扩/缩容，要么扩/缩容已经进行完了，该字典里所有的k/v都在 dict->ht[0] 上
- 不稳定字典 说的是 正在进行 rehash的字典，扩/缩容尚未完全结束，该字典里所有的k/v，一部分在dict->ht[0]上，一部分在dict->ht[1]上，且部分k/v会从 dict->ht[0] 重新映射到 dict->ht[1]


## 字典的两种不稳定情况
让我们先来看下字典有几种不稳定情况？

贴一下hash表结构
```javascript
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
### 1.字典扩容，由小变大
这种情况下，字典的k/v分别存放在 dict->ht[0]，dict->ht[1]中，
且dict->ht[1]的 大小和哈希表大小掩码 都`大于` dict->ht[0]的。

### 2.字典缩容，由大变小
这种情况下，字典的k/v分别存放在 dict->ht[0]，dict->ht[1]中，
且dict->ht[1]的 大小和哈希表大小掩码 都`小于` dict->ht[0]的。

## 1个容易想到的遍历方法
我们记录一下当前遍历字典的状态。
比如 记录 该字典下 dict->ht[0], dict->ht[1]的状态，比如每个哈希表中的元素个数之类的，
正在遍历哪个dict->ht，以及相应的游标。

再下一次遍历的时候，对比上次状态，如果状态发生改变，那我们就重新开始遍历一下....
是的，这样暴力遍历，能不遗漏k/v，但是会造成较大的k/v重复，且遍历效率低下。
如果每次快遍历完了，状态变了... 就又要重头来过，是不是有点费劲 🙃️

## 不用重新从头遍历的技巧
redis dictScan 的遍历技巧
1. 假设游标从0遍历到了v，
字典中两个hash表 用 htmin 表示较小的表, htmax 表示较大的表
2. 其中 htmin的大小掩码 是m0, htmax的大小掩码 是m1 
3.  游标v在htmin表中的下标是 v&m0，记为 idx0
    在htmax表中的下标是 v&m1 , 记为 idx1
4. 每次遍历的最小单位是一个 桶里的所有元素， (这里的桶就是哈希表数组下标)
遍历的本次游标和下次游标之间的所有桶。
6. 不管是扩容还是缩容之后， m0,m1,idx0,idx1怎样变化，如果我们都能保证
 `htmin[i],(i<idx0), htmax[j],(j<idx1)中的所有的元素都是被遍历过的`
 那当字典状态发生改变时，我们就不需要再冲头开始遍历了。
 并且我们也不再需要在迭代器里记录字典状态，只需要记录是 字典名和游标就可以了。

### 字典扩容情况的遍历
那上面保证的这个规则怎么实现呢？让我们来举个例子。


假设 dict->ht[0] 大小为 16 , 二进制掩码为 1111 记为 m0
key1 的hash值为 21 二进制为  10101，
当前游标为 5， 二进制为 0101 记为 v

 > 下标、游标与游标计算 的小技巧
 > `下标与游标在逻辑上把其二进制的高低位反转`
 > `且计算下一次游标时根据大表掩码长度反转游标，从反转后的低位+1`

发生扩容的情况
1. key1 在dict->ht[0]中的下标是 5，二进制是 0101          
    > 5 = 0101 = 哈希值 & 掩码 =  10101 & 1111
2. 本次遍历前，dict发生扩容  dict->ht[1] 大小为 32, 二进制掩码为 11111 记为m1
3. 先遍历 小表.table[5] (dict->ht[0]) 中的所有元素，key1将被遍历到。
   >  5 = 0101 = 游标 & 掩码m0 = 0101 & 1111

 4. 再遍历大表 dict->ht[1]中的一些元素👇
    >  因为在此之前 dict->ht[0].table[5]中的部分元素可能已经映射到了 dict->ht[1]中
    > 因为  1111 ^ 11111 = 10000, 在dict->ht[0].table[5]中的所有元素，
    > 映射到dict->ht[1]中，下标只可能是  5 (00101) 和 21 (10101) 
    > 所以 dict->ht[1].table[5]、dict->ht[1].table[21]  需要遍历
   
  5. 使用游标遍先遍历大表.table[5] (dict->ht[1])
     >  5 = 0101 = 游标 & 掩码m1 = 0101 & 11111
  6.  `游标 5 二进制 0101 ，按大表表掩码 11111 的长度 将二进制位反转+1之后再反转回来， 生成下一个游标  21 二进制为 10101`
      >  00101 反转-> 10100 +1-> 10101 反转-> 10101
  8. 如此循环遍历大表，直到  v & (m0 ^ m1) == 0 , 本次遍历结束时 游标 v 的值为  13 二进制 01101
     > m0 ^ m1 = 1111 ^ 11111 = 10000
     > v & ( m0 ^ m1) = 01101 & 10000 = 0 终止循环
     > 也就是将本次传入的游标与下一次传入的游标之间的所有桶，都遍历了一遍，后面会解释为什么会这样。

4. 假设本次遍历后，kye1 被重新映射到 dict->ht[1].table[21] 中 21对应的二进制位 10101
    > 21 = 10101 = 哈希值 & 掩码m1 = 10101 & 11111
5. `如果将游标的高低位颠倒过来，从左边表示高位，右边表示低位 变成 左边低位，右边表示高位`
6. `那么 key1在 dict->ht[1]中的下标 10101 是 “小于” 新的游标 1101 的！`
`那在之后的遍历中 key1将不会再被遍历到`

发生扩容情况的小结
1. 不会重复遍历
所有在dict->ht[0]中被遍历过的 key，映射到 dict->ht[1]后，都不会再被遍历到。
可以想一想为什么 🙃️
   > 在逻辑上 将游标的高低位颠倒过来表示，从左边高位，右边低位 变成 左边低位，右边高位
   > `按大掩码m1的长度 对 游标进行反转, +1, 再反转的操作`，可以理解为`"单调递增"`。
   > 而已经遍历过的key，它的hash值是不会变的！
   > 已经被遍历过的key，不管是在dict->ht[0]，还是在 dict->ht[1]中的下标
   > 大小按高低位反转来算 都会小于 下一次游标的值
   > 所以不会再被遍历到


9. 遍历没有遗漏的k/v
对所有未被遍历到的key的hash值不管是在 dict->ht[0]，还是在 dict->ht[1]中对应的下标，
`下标与游标的大小按二进制高低位反转来算`，值都大于等于当前游标，都会被遍历到。
因为游标在"单调递增"啊
并且对游标v的遍历中，将本次游标与下次游标之间的桶都遍历了

   也就是说 下标大于游标的桶，在将来会被遍历
   下标 介于本次游标和下次游标之间的，在本次被遍历
   下标 小于本次游标的，在之前已经被遍历过了
 
  

所以当字典扩容时，该方法既不会漏掉k/v也不会重复遍历。

> 当字典扩容时，如果我们对两个hash表 按下标从0开始+1遍历，
虽然不会漏掉key，但对一些key是会重复遍历的，
可以想一想为什么🤔️

### 字典缩容情况的遍历
字典缩容情况下，虽然遍历不会遗漏k/v，但是会存在部分重复遍历的情况，
但该重复量 相对暴力遍历来说少了不少。

重复遍历情况举例:
1. dict->ht[0] 的大小为 16，掩码 1111，本次遍历游标为 5 (0101)
2. key1 的hash值为 21 (10101) 在dict->ht[0]中的下标为 5 (0101)
3. 本次遍历 key1将被遍历到，本次遍历结束计算下次遍历游标  13 (1101)
4. 下一次遍历游标13前， 
字典发生缩容， dict->ht[1] 的大小为4 掩码为 11
key1 被映射到 dict->ht[1].table[1]    
   > (1 = 10101 & 11)
6. 使用游标13 (1101)继续遍历
7. dict->ht[1].table[1] 将被遍历 ( 1 = 1101 & 11 )
8. key1被重复遍历了

缩容情况下，遍历不会遗漏k/v，可以想想为什么。

# 小结
redis不稳定字典的遍历，主要是使用 
`将哈希数组下标与游标在逻辑上把其二进制的高低位反转，且计算下一次游标时根据大表掩码长度反转游标，从反转后的低位+1的技巧，利用 游标 "单调递增"的性质。遍历时不遗漏 k/v ，扩容时不会重复遍历，缩容可能有部分重复遍历。`


思考题🤔
1. 前后两次迭代中间，如果经历了 n多次扩容/缩容.... 这种遍历方式还能很好的支持吗？为什么？
2. 下标、游标的大小表示按正常高低位表示，游标每次加1这种操作，会有什么问题?
# 参考资料
[Redis源码解析：04字典的遍历dictScan](https://blog.csdn.net/gqtcgq/article/details/50533336?locationNum=6&fps=1)
# 彩蛋
dictScan最初是由 Pieter Noordhuis提交的代码，
[redis作者antirez 与Pieter Noordhuis的在线讨论](https://github.com/antirez/redis/pull/579)

 # 往期博客回顾
1. [redis服务器的部分启动过程](https://blog.csdn.net/a158372582/article/details/106023071)
2.  [GET命令背后的源码逻辑](https://editor.csdn.net/md/?articleId=106035658)
3. [redis的基础数据结构之 sds](https://blog.csdn.net/a158372582/article/details/106063645)
4. [redis的基础数据结构之 list](https://blog.csdn.net/a158372582/article/details/106086284)
5. [redis的基础数据结构 之 ziplist](https://blog.csdn.net/a158372582/article/details/106107759)
6. [redis 基础数据结构之 hash表](https://blog.csdn.net/a158372582/article/details/106234075)


# dictScan源码与注释
这是 redis 6.0中的部分源码
```javascript
/* dictScan() is used to iterate over the elements of a dictionary.
 *
 * Iterating works the following way:
 *
 * 1) Initially you call the function using a cursor (v) value of 0.
 * 2) The function performs one step of the iteration, and returns the
 *    new cursor value you must use in the next call.
 * 3) When the returned cursor is 0, the iteration is complete.
 *
 * The function guarantees all elements present in the
 * dictionary get returned between the start and end of the iteration.
 * However it is possible some elements get returned multiple times.
 *
 * For every element returned, the callback argument 'fn' is
 * called with 'privdata' as first argument and the dictionary entry
 * 'de' as second argument.
 *
 * HOW IT WORKS.
 *
 * The iteration algorithm was designed by Pieter Noordhuis.
 * The main idea is to increment a cursor starting from the higher order
 * bits. That is, instead of incrementing the cursor normally, the bits
 * of the cursor are reversed, then the cursor is incremented, and finally
 * the bits are reversed again.
 *
 * This strategy is needed because the hash table may be resized between
 * iteration calls.
 *
 * dict.c hash tables are always power of two in size, and they
 * use chaining, so the position of an element in a given table is given
 * by computing the bitwise AND between Hash(key) and SIZE-1
 * (where SIZE-1 is always the mask that is equivalent to taking the rest
 *  of the division between the Hash of the key and SIZE).
 *
 * For example if the current hash table size is 16, the mask is
 * (in binary) 1111. The position of a key in the hash table will always be
 * the last four bits of the hash output, and so forth.
 *
 * WHAT HAPPENS IF THE TABLE CHANGES IN SIZE?
 *
 * If the hash table grows, elements can go anywhere in one multiple of
 * the old bucket: for example let's say we already iterated with
 * a 4 bit cursor 1100 (the mask is 1111 because hash table size = 16).
 *
 * If the hash table will be resized to 64 elements, then the new mask will
 * be 111111. The new buckets you obtain by substituting in ??1100
 * with either 0 or 1 can be targeted only by keys we already visited
 * when scanning the bucket 1100 in the smaller hash table.
 *
 * By iterating the higher bits first, because of the inverted counter, the
 * cursor does not need to restart if the table size gets bigger. It will
 * continue iterating using cursors without '1100' at the end, and also
 * without any other combination of the final 4 bits already explored.
 *
 * Similarly when the table size shrinks over time, for example going from
 * 16 to 8, if a combination of the lower three bits (the mask for size 8
 * is 111) were already completely explored, it would not be visited again
 * because we are sure we tried, for example, both 0111 and 1111 (all the
 * variations of the higher bit) so we don't need to test it again.
 *
 * WAIT... YOU HAVE *TWO* TABLES DURING REHASHING!
 *
 * Yes, this is true, but we always iterate the smaller table first, then
 * we test all the expansions of the current cursor into the larger
 * table. For example if the current cursor is 101 and we also have a
 * larger table of size 16, we also test (0)101 and (1)101 inside the larger
 * table. This reduces the problem back to having only one table, where
 * the larger one, if it exists, is just an expansion of the smaller one.
 *
 * LIMITATIONS
 *
 * This iterator is completely stateless, and this is a huge advantage,
 * including no additional memory used.
 *
 * The disadvantages resulting from this design are:
 *
 * 1) It is possible we return elements more than once. However this is usually
 *    easy to deal with in the application level.
 * 2) The iterator must return multiple elements per call, as it needs to always
 *    return all the keys chained in a given bucket, and all the expansions, so
 *    we are sure we don't miss keys moving during rehashing.
 * 3) The reverse cursor is somewhat hard to understand at first, but this
 *    comment is supposed to help.
 */
 #define dictIsRehashing(d) ((d)->rehashidx != -1)

unsigned long dictScan(dict *d,
                       unsigned long v,
                       dictScanFunction *fn,// key/value的回调处理函数
                       dictScanBucketFunction* bucketfn,//桶的回调处理函数
                       void *privdata)
{
    dictht *t0, *t1;
    const dictEntry *de, *next;
    unsigned long m0, m1;
	// 跳过空字典
    if (dictSize(d) == 0) return 0;

    /* Having a safe iterator means no rehashing can happen, see _dictRehashStep.
     * This is needed in case the scan callback tries to do dictFind or alike. */
    d->iterators++;

    if (!dictIsRehashing(d)) {//稳定字典
        t0 = &(d->ht[0]);
        m0 = t0->sizemask;

        /* Emit entries at cursor */
        // 桶回调处理函数
        if (bucketfn) bucketfn(privdata, &t0->table[v & m0]);
        de = t0->table[v & m0];
        while (de) {
            next = de->next;
            // key/value的回调处理函数
            fn(privdata, de);
            de = next;
        }

        /* Set unmasked bits so incrementing the reversed cursor
         * operates on the masked bits */
        v |= ~m0; //根据掩码m0的长度来算v

        /* Increment the reverse cursor */
        v = rev(v);//反转
        v++;	   // +1
        v = rev(v); //反转

    } else { //不稳定字典
        t0 = &d->ht[0];
        t1 = &d->ht[1];

        /* Make sure t0 is the smaller and t1 is the bigger table */
        //无视 是扩容还是缩容，按小表到大表的顺序遍历
        if (t0->size > t1->size) {
            t0 = &d->ht[1];
            t1 = &d->ht[0];
        }
		
        m0 = t0->sizemask;
        m1 = t1->sizemask;

        /* Emit entries at cursor */
        // 桶的回调处理函数
        if (bucketfn) bucketfn(privdata, &t0->table[v & m0]);
        de = t0->table[v & m0];
        while (de) {
            next = de->next;
            // key/value 的回调处理函数
            fn(privdata, de);
            de = next;
        }

        /* Iterate over indices in larger table that are the expansion
         * of the index pointed to by the cursor in the smaller table */
        do {
            /* Emit entries at cursor */
            // 桶的回调处理函数
            if (bucketfn) bucketfn(privdata, &t1->table[v & m1]);
            de = t1->table[v & m1];
            while (de) {
                next = de->next;
                // key/value 的回调处理函数
                fn(privdata, de); 
                de = next;
            }

            /* Increment the reverse cursor not covered by the smaller mask.*/
            v |= ~m1;	//按大表掩码长度来算下一个游标
            v = rev(v); //反转
            v++;		// +1
            v = rev(v); // 反转

            /* Continue while bits covered by mask difference is non-zero */
        } while (v & (m0 ^ m1)); // 将本次传入的游标，与下次迭代的游标之间的所有桶，全部遍历
    }

    /* undo the ++ at the top */
    d->iterators--;

    return v;
}

 /* Function to reverse bits. Algorithm from:
 * http://graphics.stanford.edu/~seander/bithacks.html#ReverseParallel */
 * 二进制反转，先反转一半，再反转一半的一半...然后就全部反转完了  🙃️
 * 写法有点骚气
static unsigned long rev(unsigned long v) { 
    unsigned long s = 8 * sizeof(v); // bit size; must be power of 2
    unsigned long mask = ~0;
    while ((s >>= 1) > 0) {
        mask ^= (mask << s);
        v = ((v >> s) & mask) | ((v << s) & ~mask);
    }
    return v;
}
```