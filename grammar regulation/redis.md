https://www.runoob.com/redis/redis-data-types.html

 

HSCAN key cursor [MATCH pattern] [COUNT count]

hscan pms:1 0 match stock:* count 100

当SCAN命令的游标参数被设置为 0 时， 服务器将开始一次新的迭代

 

Hgetall 以列表形式返回哈希表的字段及字段值

 

查看有多少键：keys *



基本类型： 

**List、set、hash、zset、string**



ZADD KEY_NAME SCORE1 VALUE1.. SCOREN VALUEN

ZADD myzset 1 "one"

 

 

Redis用crc16进行hash

没有用一致性hash(对2^32次mod、数据点倾斜则用虚拟节点，虚拟节点到实际节点映射即可)

 

Redis的瓶颈最有可能是机器内存的大小或者网络带宽

Redis为什么这么快

1.完全基于内存

2.数据结构简单，对数据的操作简单，redis中的ds是专门设计过的

3.单线程，避免不必要的上下文切换和竞争条件

4.Redis 是单线程+多路IO复用技术

5.非阻塞I/O多路复用机制

单个线程高效的处理多个连接请求，利用select、poll、epoll可以监察多个流的I/O事件的能力。Select轮询max1024、poll



zset底层：

一、ziplist（双向链表，节省内存空间。查找是O(n)）使用条件：1、保存的元素数量小于128个  	2、保存的所有元素的长度小于64字节

保存有序的元素列表。用两个紧挨在一起的压缩列表节点来保存，第一个保存元素成员，第二个保存元素的分值。zlbytes，表示整个list的总长度，zltail最后一个元素在ziplist的偏移。zllen表示元素个数

![image-20210103164611717](H:\javawork\1230\Java_bfund\grammar regulation\image-20210103164611717.png)

二、skiplist，包括一个dict（key->score）对象和skiplist(score->key的映射)对象。dict存key/value（key元素value分值）。

![img](https://upload-images.jianshu.io/upload_images/6302559-7cdb7b7aeebec44b.png?imageMogr2/auto-orient/strip|imageView2/2/format/webp)

```cpp
/*
 * 有序集合
 */
typedef struct zset {

    // 字典，键为成员，值为分值
    // 用于支持 O(1) 复杂度的按成员取分值操作
    dict *dict;

    // 跳跃表，按分值排序成员
    // 用于支持平均复杂度为 O(log N) 的按分值定位成员操作
    // 以及范围操作
    zskiplist *zsl;

} zset;
```

```cpp
/*
 * 跳跃表
 */
typedef struct zskiplist {

    // 表头节点和表尾节点
    struct zskiplistNode *header, *tail;

    // 表中节点的数量
    unsigned long length;

    // 表中层数最大的节点的层数
    int level;

} zskiplist;
```

skiplist跳表中节点格式，每个节点保存数据的robj，分值score字段，backward便于回溯，zskiplistlevel的数组保存跳跃列表每层的节点数

```cpp
/*
 * 跳跃表节点
 */
typedef struct zskiplistNode {

    // 成员对象
    robj *obj;

    // 分值
    double score;

    // 后退指针
    struct zskiplistNode *backward;

    // 层
    struct zskiplistLevel {

        // 前进指针
        struct zskiplistNode *forward;

        // 跨度，到达该层下一个节点实际(level[0])跨越了多少个节点
        unsigned int span;

    } level[];

} zskiplistNode;
```

zadd底层操作

1、如果是ziplist，存在则先删除后添加，不存在需要考虑总长度是否会大于64字节，数量是否小于128个。否则会转化成skiplist。

2、如果是skiplist，存在则先删除再添加，存在则直接添加，并且需要再dict那更新。

查找，类似二分（空间换时间），时间复杂度O(logn)，分层，从最高层往下降去查找。L0层会有所有元素的有序排列。节点的level是随机的，level越底层，比重越高。

zadd key score1 member1 score2 member2

