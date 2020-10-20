---
title: "Redis：数据结构"
date: 2020-10-09T18:17:27+08:00
slug: ""
description: "介绍Redis几种数据结构, 字符串, 列表, 字典, 集合, 有序集合, 跳表"
keywords: [Redis, 字符串, 列表, 字典, 集合, 有序集合, 跳表]
tags: [Redis]
math: false
toc: true
---

Redis 是一个使用 ANSI C 编写的开源、支持网络、基于内存、可选持久性的键值对存储数据库。从 2015 年 6 月开始，Redis 的开发由 Redis Labs 赞助，而 2013 年 5 月至 2015 年 6 月期间，其开发由 Pivotal 赞助。在 2013 年 5 月之前，其开发由 VMware 赞助。根据月度排行网站 DB-Engines.com 的数据，Redis 是最流行的键值对存储数据库。本文将介绍 Redis 的数据结构及其底层实现。

Redis 中键的类型只能为字符串，值支持多种数据类型：

* 字符串 - string；
* 列表 - list；
* 字典 - hash； 
* 集合 - set；
* 有序集合 - sorted set；
* 其他还有范围查询，bitmap，hyperloglog 和地理空间（geospatial）索引半径查询。

## 一、简单动态字符串

Redis 没有直接使用 C 语言传统的字符串表示（以空字符结尾的字符数组，以下简称 C 字符串），而是自己构建了一种名为**简单动态字符串**（**S**imple **D**ynamic **S**tring，**SDS**）的抽象类型，并将 SDS 用作 Redis 的默认字符串表示。每一个 *sds.h/sdshdr* 结构体表示一个 SDS：

```c
struct sdshdr {
    
    // 保存字符串的长度
    int len;					// 5
    
    // buf 中未使用的数量				
    int free;					// 5
    
    // 保存字符串的字符数组
    char buf[];					// 'r', 'e', 'd', 'i', 's', _, _, _, _, _, '\0'
    
}
```

虽然 SDS 记录了每个字符串的长度，并不需要结尾加 '\0' 来表示结束，但是为了兼容 C 语言的字符串 API，还是保留了 '\0' 这个结尾。

### 1.1 SDS 与 C 字符串的区别

C 字符串实际上是使用 null 字符 '\0' 终止的一维字符数组。SDS 相较之下有以下优势：

#### 1. 常数复杂度获取字符串长度

因为 C 字符串并不记录自身的长度信息，所以为了获取一个 C 字符串的长度，程序必须遍历整个字符串，对遇到的每个字符进行计数，直到遇到代表字符串结尾的空字符为止，这个操作是线性复杂度的。而 SDS 只需要读取 *len* 字段，属于常数级复杂度。

#### 2. 杜绝缓存区溢出

C 字符串在使用 `strcat(s1, s2)` 拼接时，如果没有为 *s1* 分配足够的内存，就会出现其他数据被意外修改为 *s2* 中字符的情况。而 SDS 在进行字符串拼接时会先确定内存是否足够，如果不够就会先扩展 SDS 的空间。

#### 3.  减少修改字符串带来的内存重分配次数

修改 SDS 时，通过**空间预分配**和**惰性空间释放**，可以减少内存重分配的次数。

空间预分配用于优化 SDS 的字符串**增长**操作：当 SDS 的 API 对一个 SDS 进行修改，并且需要对 SDS 进行空间扩展的时候，程序不仅会为 SDS 分配修改所必须要的空间，还会为 SDS 分配额外的未使用空间，额外空间大小（即 *free* 的大小）等于增长后的 *len*，但不超过 1MB。

惰性空间释放用于优化 SDS 的字符串**缩短**操作：当 SDS 的 API 需要缩短 SDS 保存的字符串时，程序并不立即使用内存重分配来回收缩短后多出来的字节，而是使用 *free* 属性将这些字节的数量记录起来，并等待将来使用。当然 SDS 也提供了相应的 API 用来真正地释放 SDS 的未使用空间，所以不用担心惰性空间释放策略会造成内存浪费。

### 1.2 二进制安全

C 字符串并不是二进制安全的，所谓二进制安全就是任意二进制序列都不存在特殊含义，而 C 字符串中 '\0' 表示字符串终止。如果二进制数据中包括 '\0'，将其当作 C 字符串来处理的话就会提前截断，导致信息的丢失。因此，C 字符串不能用来储存图片、音频、视频和压缩文件这样的二进制数据。

SDS 可以解决这个问题，原因就在于 SDS 通过记录 *len* 来标识哪里终止。*buf* 中的任意二进制序列都不表示特殊的含义。

## 二、链表

链表（List）提供了高效的节点重排能力，以及顺序性的节点访问方式，并且可以通过增删节点来灵活地调整链表的长度。

每个链表节点使用一个 *adlist.h/listNode* 结构表示：

```c
typedef struct listNode {
    
    // 前置节点
    struct listNode *prev;
    
    // 后置节点
    struct listNode *next;
    
    // 节点的值
    void *value;
    
} listNode;
```

就像 Java 中的 *Deque*，创建一个链表通常会定义头节点和尾节点，这样在增删操作时就不需要判断是否是头尾节点，更加方便。

```c
typedef struct list {
    
    // 头节点
    listNode *head;
    
    // 尾节点
    listNode *tail;
    
    // 链表所包含的节点数
    unsigned long len;
    
    // 节点值复制函数
    void *(*dup) (void *ptr);
    
    // 节点值释放函数
    void (*free) (void *ptr);
    
    // 节点值对比函数
    int (*match) (void *ptr, void *key);
    
} list;
```

Redis 的链表实现的特性可以总结如下：

* 双端：链表节点有 *prev* 和 *next* 指针，获取前置节点和后置节点的复杂度都是 O(1）；
* 无环：头节点的 *prev* 指针和尾节点的 *next* 指针都指向 null，对链表访问以 null 为终点；
* 带表头指针和表尾指针：通过链表结构的 *head* 指针和 *tail* 指针，程序获取链表的表头节点和表尾节点的复杂度为 O(1）；
* 带链表长度计数器：程序获取链表中节点数量（*len*）的复杂度为 O(1）；
* 多态：链表节点使用 `void*` 指针来保存节点值，并且可以通过 *list* 结构的 *dup*、*free*、*match* 三个属性为节点值设置类型特定函数，所以链表可以用于保存各种不同类型的值。

## 三、字典

Redis 的字典使用哈希表作为底层实现，一个哈希表里面可以有多个哈希表节点，而每个哈希表节点就保存了字典中的一个键值对。这与 Java 中的 *HashMap* 逻辑基本是一致的。

哈希表节点使用 *dictEntry* 结构表示：

```c
typedef struct dictEntry {
    
    // 键
    void *key;
    
    // 值
    union {
        void *val;
        unit64_tu64;
        int64_ts64;
    } v;
    
    // 链表结构以解决哈希冲突
    struct dictEntry *next;
    
} dictEntry;
```

*key* 属性保存着键值对中的键，而 *v* 属性则保存着键值对中的值，其中键值对的值可是一个指针，或者是一个 uint64_t 整数，又或者是一个 int64_t 整数。

*next* 属性是指向另一个哈希表节点的指针，这个指针可以将多个哈希值相同的键值对接在一次，以此来解决键冲突（collision）的问题。

哈希表使用 *dict.h/dictht* 结构表示：

```c
typedef struct dictht {
    
    // 哈希表节点数组
    dictEntry **table;
    
    // 哈希表大小
    unsigned long size;
    
    // 哈希表大小掩码（size - 1），用于计算索引值（& 位运算）
    unsigned long sizemask;
    
    // 哈希表已有节点数
    unsigned long used;
    
} dictht;
```

字典使用 *dict.h/dict* 结构表示：

```c
typedef struct dict {
    
    // 类型特定函数
    dictType *type;
    
    // 私有数据
    void *privdata;
    
    // 哈希表
    dictht ht[2];
    
    // rehash 索引，不在进行时为 -1
    int rehashidx;
    
} dict;
```

*type* 属性和 *privdata* 属性是针对不同类型的键值对，为创建多态字典而设置的。

*ht* 属性是一个包含两个 *dictht* 的数组，一般情况下，字典只使用 *ht[0]* 哈希表，*ht[1]* 哈希表只会在对 *ht[1]* 哈希表进行 `rehash()` 时使用。

*rehashidx* 记录了 `rehash()` 的进度，如果目前没有在进行 `rehash()`，那么它的值为 -1。

### 3.1 哈希算法

和 Java 中的 *HashMap* 相似，字典先通过某种哈希算法获得键的哈希值，然后通过哈希值与掩码 *sizemask* 做与位运算来获得桶下标：

```c
index = hash & sizemask;
```

Redis 中使用的哈希算法是 MurmurHash，MUR 表示 MUltiply and Rotate，MurmurHash 在哈希的过程要经过多次 MUltiply and Rotate，因此得名。

> MurmurHash 是一种非加密型哈希函数，适用于一般的哈希检索操作。由Austin Appleby在2008年发明，与其它流行的哈希函数相比，对于规律性较强的 key，MurmurHash 的随机分布特征表现更良好。

### 3.2 渐进式 rehash

和 Java 中的 *HashMap* 相似，Redis 的字典也使用链地址法来解决哈希冲突。不同的是，Redis 采用头插法，而 JDK 1.8 由于需要转换红黑树等原因采用尾插法。

Java 中扩容时会新建一个两倍于原容量的 *newTab*。Redis 中使用一个包含两个 *dictht* 的数组，原理是相同的。除此之外 Redis 中的字典还可以收缩，当键值对数量过少时，通过收缩可以减少内存占用。具体过程如下：

1. 为字典的 *ht[1]* 哈希表分配空间，这个哈希表的空间大小取决于要执行的操作，以及 *ht[0]* 当前包含的键值对数量（*ht[0].used*）：
   * 扩展操作：第一个大于等于 *ht[0].used* * 2 的 2"（2的n次方幂）；
   * 收缩操作：第一个大于等于 *ht[0].used* 的 2"；
2. 将保存在 *ht[0]* 中的所有键值对 rehash 到 *ht[1]* 上面：rehash 指的是重新计算键的哈希值和索引值，然后将键值对放置到 *ht[1]* 哈希表的指定位置上；
3. 当 *ht[0]* 包含的所有键值对都迁移到了 *ht[1]* 之后（*ht[0]* 变为空表），释放 *ht[0]*，将 *ht[1]* 设置为 *ht[0]*，并在 *ht[1]* 新创建一个空白哈希表，为下一次 rehash 做准备。

通常 Redis 中储存的数据量是很大的，为了避免 rehash 对服务器性能造成影响，服务器不是一次性将 *ht[0]* 里面的所有键值对全部 rehash 到 *ht[1]*，而是分多次、渐进式地将 *ht[0]* 里面的键值对慢慢地 rehash到 *ht[1]*，具体步骤如下：

1. 为 *ht[1]* 分配空间，让字典同时持有 *ht[0]* 和 *ht[1]* 两个哈希表；
2. 字典中维持一个索引计数器 *rehashidx*，并将它的值设置为 0，表示 rehash 工作正式开始；
3. 在 rehash 进行期间，每次对字典执行添加、删除、查找或者更新操作时，程序除了执行指定的操作以外，还会顺带将 *ht[0]* 哈希表在 *rehashidx* 索引上的所有键值对 rehash 到 *ht[1]*，当 rehash 工作完成之后，程序将 *rehashidx* 属性的值增一；
4. 随着字典操作的不断执行，最终在某个时间点上，*ht[0]* 的所有键值对都会被 rehash 至 *ht[1]*，这时程序将 *rehashidx*属性的值设为 -1，表示 rehash 操作已完成。

## 四、集合 

Redis 的集合相当于 Java 语言中的 *HashSet*，它内部的键值对是无序、唯一的。它的内部实现相当于一个特殊的字典，字典中所有的 value 都是一个值 null。

整数集合（intset）是集合键的底层实现之一，当一个集合只包含整数值元素，并且这个集合的元素数量不多时， Redis 就会使用整数集合作为集合键的底层实现。

*intset.h/intset* 结构表示一个整数集合，其中的 *contents* 数组是整数集合的底层实现：整数集合的每个元素都是 *contents* 数组的一个数组项（item），各个项在数组中按值的大小从小到大有序地排列，并且数组中不包含任何重复项。

## 五、有序集合

Redis 使用跳跃表作为有序集合键的底层实现之一，如果一个有序集合包含的元素数量比较多，又或者有序集合中元素的成员（member）是比较长的字符串时， Redis 就会使用跳跃表来作为有序集合键的底层实现。

跳跃表（skiplist）是一种有序数据结构，它通过在每个节点中维持多个指向其他节点的指针，从而达到快速访问节点的目的。

跳跃表支持平均 O(logM）、最坏 O(M) 复杂度的节点査找，原理可见 [[2]](https://zhuanlan.zhihu.com/p/53975333)，还可以通过顺序性操作来批量处理节点。在大部分情况下，跳跃表的效率可以和平衡树相媲美，并且因为跳跃表的实现比平衡树要来得更为简单，所以有不少程序都使用跳跃表来代替平衡树。

当使用 REDIS_ENCODING_SKIPLIST 编码时， 有序集元素由 *redis.h/zset* 结构来保存：

```c
typedef struct zset {

    // 字典
    dict *dict;

    // 跳跃表
    zskiplist *zsl;

} zset;
```

*zset* 同时使用字典和跳跃表两个数据结构来保存有序集元素。

其中， 元素的成员由一个 *redisObject* 结构表示， 而元素的 *score* 则是一个 *double* 类型的浮点数， 字典和跳跃表两个结构通过将指针共同指向这两个值来节约空间 （不用每个元素都复制两份）。

## 参考

1. 黄健宏. Redis设计与实现[M]. 机械工业出版社, 2014.
2. [图解跳表](https://zhuanlan.zhihu.com/p/53975333)