---
attachments: [Clipboard_2021-05-23-18-38-23.png, Clipboard_2021-05-23-19-00-26.png, Clipboard_2021-05-23-19-12-26.png, Clipboard_2021-05-23-19-36-03.png, Clipboard_2021-05-23-23-11-31.png, Clipboard_2021-05-24-23-50-32.png, Clipboard_2021-05-25-00-44-18.png, Clipboard_2021-05-25-20-54-12.png, Clipboard_2021-05-25-21-38-25.png, Clipboard_2021-05-26-22-46-18.png, Clipboard_2021-05-27-00-04-16.png, Clipboard_2021-05-27-00-36-32.png, Clipboard_2021-05-27-00-52-39.png]
title: Redis 基础知识
created: '2021-05-20T12:14:49.968Z'
modified: '2021-05-26T16:57:24.562Z'
---

# Redis 基础知识

## Redis 介绍(百度)
redis是一个key-value存储系统。和Memcached类似，它支持存储的value类型相对更多，包括string(字符串)、list(链表)、set(集合)、zset(sorted set --有序集合)和hash（哈希类型）。这些数据类型都支持push/pop、add/remove及取交集并集和差集及更丰富的操作，而且这些操作都是原子性的。在此基础上，redis支持各种不同方式的排序。与memcached一样，为了保证效率，数据都是缓存在内存中。区别的是redis会周期性的把更新的数据写入磁盘或者把修改操作写入追加的记录文件，并且在此基础上实现了master-slave(主从)同步。
Redis 是一个高性能的key-value数据库。 redis的出现，很大程度补偿了memcached这类key/value存储的不足，在部分场合可以对关系数据库起到很好的补充作用。它提供了Java，C/C++，C#，PHP，JavaScript，Perl，Object-C，Python，Ruby，Erlang等客户端，使用很方便。
Redis支持主从同步。数据可以从主服务器向任意数量的从服务器上同步，从服务器可以是关联其他从服务器的主服务器。这使得Redis可执行单层树复制。存盘可以有意无意的对数据进行写操作。由于完全实现了发布/订阅机制，使得从数据库在任何地方同步树时，可订阅一个频道并接收主服务器完整的消息发布记录。同步对读取操作的可扩展性和数据冗余很有帮助。

## Redis 对象
Redis的key只是字符串类型，但是value却支持多种对象类型，`server.h`中redisObject的结构定义为：
```c
typedef struct redisObject {
    unsigned type:4; // 4bits，对象类型
    unsigned encoding:4; // 4bits，对象存储格式
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time).(3bytes) */
    int refcount; // 32bits(4bytes)，对象引用计数，为0时被回收。
    void *ptr; // 对象地址指针。8bytes=64位系统。不考虑32位系统。
} robj;
```
一个 RedisObject 对象头共需要占据 16 字节的存储空间。
注：均*基于Redis6.2.3版本*源码分析。

| 数据类型 | type取值 | 说明 |
| - | - | - |
| OBJ_STRING | 0 | 字符串对象 (实际可以为整型，浮点型以及字符串) |
| OBJ_LIST | 1 | 列表对象（实际队列，元素可以重复，支持FIFO原则，对比Java的List） |
| OBJ_SET | 2 | set (不可重复的集合，对比Java的Set) |
| OBJ_ZSET | 3 | sorted set (有序集合) |
| OBJ_HASH | 4 | hash (Hash表) |
| OBJ_MODULE | 5 | 模块对象（在RDB文件中的编码形式为，具有64位的模块ID，其中54位为模块特定签名，10位为编码版本） |
| OBJ_STREAM | 6 | 流对象（5.0版本以后新增的，为了更好的当做消息队列使用）

---

| 编码类型 | encoding取值 | 说明 | Object encoding命令输出 |
| - | - | - | - |
| OBJ_ENCODING_RAW | 0 | 简单动态字符串 | raw |
| OBJ_ENCODING_INT | 1 | long类型的整数 | int |
| OBJ_ENCODING_HT | 2 | 字典 | hashtable |
| OBJ_ENCODING_ZIPMAP | 3 | 压缩字典 | 无了 |
| OBJ_ENCODING_LINKEDLIST | 4 | 双端链表 | 无了 | 
| OBJ_ENCODING_ZIPLIST | 5 | 压缩列表 | ziplist |
| OBJ_ENCODING_INTSET | 6 | 整数集合 | intset |
| OBJ_ENCODING_SKIPLIST | 7 | 跳跃表 | skiplist |
| OBJ_ENCODING_EMBSTR | 8 | embstr编码的简单动态字符串 | embstr |
| OBJ_ENCODING_QUICKLIST | 9 | 快速列表 | quicklist |
| OBJ_ENCODING_STREAM | 10 | 流 | stream |

### 字符串对象（String）
字符串对象支持的存储格式有3种：
> - OBJ_ENCODING_INT：当存储的是数值类型的时。
> - OBJ_ENCODING_EMBSTR：当存储的是字符串且长度小于等于44字节时。
> - OBJ_ENCODING_RAW：当存储的是字符串且长度大于44字节时。

```c
struct __attribute__ ((__packed__)) sdshdr[5|8|16|32|64] {
    uint[8|16|32|64]_t len; // 已使用的长度 (为5时没有)
    uint[8|16|32|64]_t alloc; // 分配的总长度 (为5时没有)
    unsigned char flags; // 3bits标识，5bits没有用。5=000，8=001，16=010，32=011,64=100。
    char buf[];
};
```
SDSHDR的结构大致如下：
![SDSHDR的结构](@attachment/Clipboard_2021-05-23-19-00-26.png)
> flags=0的场景有待验证，从源码中看是定长，与C字符串类似，可能用作常量？

横向表示为内存地址空间。
#### OBJ_ENCODING_INT
如果要存储一个数值为`100`的字符串进去
INT示意图：
![INT示意图](@attachment/Clipboard_2021-05-23-18-38-23.png)
> 1. 数值类型的值，直接存储在*ptr指针所指向的内存，没有额外的空间引用，不需要使用SDSHDR结构。

#### OBJ_ENCODING_EMBSTR
如果要存储一个`Hello`进去(小于44字节)。
```c
#define OBJ_ENCODING_EMBSTR_SIZE_LIMIT 44 // 定义为44bytes长度。
// 计算公式如下：robj=16bytes,sdshdr8=3bytes,char[]=44bytes,补齐1bytes。
robj *o = zmalloc(sizeof(robj)+sizeof(struct sdshdr8)+len+1);
```

EMBSTR示意图：
![EMBSTR示意图](@attachment/Clipboard_2021-05-23-19-12-26.png)
> 1. 当字符串长度小于等于44字节时，会以EMBSTR编码的SDSHDR类型存储。
> 2. SDSHDR的内存分配是与REDIS_OBJECT一起申请的，所以它们是连续的空间内存，**一次内存分配**即可完成。
> 3. EMBSTR类型一般不修改，如果要修改，要么长度仍小于44字节重建，要么长度大于44字节类型变换为RAW。

#### OBJ_ENCODING_RAW
如果存储一个大于44bytes的字符串`abcd...`(省略)。
RAW字符串示意：
![RAW字符串示意](@attachment/Clipboard_2021-05-23-19-36-03.png)
> 1. 当字符串长度大于44字节时，会以RAW编码的SDSHDR类型存储。
> 2. SDSHDR的内存分配是单独的一次，这是和EMBSTR不一样的地方，需要**两次内存分配**才行，不连续。
> 3. 预留空间充足时，字符串改变无需重写分配内存，不够时需要重建。

### 列表对象（List）
列表对象可支持的存储格式有2种：
> - OBJ_ENCODING_ZIPLIST：当列表中对象少（小于128个），仅有INT或短字符串(小于64bytes)时，使用压缩列表。
> - OBJ_ENCODING_QUICKLIST：当列表中的对象多，或字符串类型长，变动频繁时，用快速压缩列表。

#### OBJ_ENCODING_ZIPLIST
ZIPLIST的结构定义如下：
```c
typedef struct zlentry {
    // prevrawlensize prevrawlen通常会被压缩，小于255的用1个字节来表示，大于等于255的用5个字节表示
    unsigned int prevrawlensize; // 编码前驱节点的所需的字节大小
    unsigned int prevrawlen;     // 前驱节点的内容大小

    // lensize len通常也会被压缩，与preraw同理。
    unsigned int lensize;        // 编码当前节点的所需的字节大小
    unsigned int len;            // 当前节点的内容大小

    unsigned int headersize;     // prevrawlensize + lensize
    unsigned char encoding;      // 对字符数组或整数的编码方式，ZIP_STR_*,不同位编码长度不一样，1~5字节；ZIP_INT_*，整数编码长度只有1字节
    unsigned char *p;            // 指向当前节点实际值的指针
} zlentry;
```
压缩列表示意：
![压缩列表](@attachment/Clipboard_2021-05-23-23-11-31.png)
![Entry节点](@attachment/Clipboard_2021-05-24-23-50-32.png)
> 1. 压缩列表本是单向链表，但是由于内存空间存储的连续性，使得可以从header向tail遍历，所以可以看做双向链表，弥补了单向链表的查询弊端，但是修改会重新分配内存空间以维护连续性，所以修改的效率不一定是O(1)，**列表越大，效率越低**。
> 2. 由于会分配一个节点会记录其前驱的长度，当节点长度小于255字节时，默认只会采用一个bytes去记录节点的前驱长度，为了节省内存，但是当有一个节点大于或等于255字节时，一个bytes不够，需要扩容为5个bytes来记录前驱节点的长度，为了不频繁更新，redis直接进行连锁更新，将后续所有节点的前驱长度记录扩容为5bytes，若后续节点都是250~253bytes，最差情况**插入头节点大于等于255，引发全部节点的内存重新分配**，删除同理。
> 3. redis为了防止发生数组抖动，一会儿扩一会儿缩这种，**不处理因为节点的变小而引发的连锁更新，防止出现反复的缩小-扩展(flapping，抖动)**。

#### OBJ_ENCODING_QUICKLIST
快速列表由压缩列表构成，类似于分段的思想，将其拆分为多组ZIPLIST。
快速列表的结构定义如下：
```c
typedef struct quicklist {
    quicklistNode *head; // 头节点指针
    quicklistNode *tail; // 尾节点指针
    unsigned long count; // 所有ZIPLIST中的entry个数，即整个QUICKLIST中的entry个数
    unsigned long len; // quicklistNode节点个数，也就是ZIPLIST的个数
    int fill : QL_FILL_BITS; // 唯一标识，补齐字节位的，实际没啥用
    unsigned int compress : QL_COMP_BITS; // 压缩程度
    unsigned int bookmark_count: QL_BM_BITS;
    quicklistBookmark bookmarks[];
} quicklist;

typedef struct quicklistNode {
    struct quicklistNode *prev; // 前驱节点
    struct quicklistNode *next; // 后继节点
    unsigned char *zl; // 指向当前ZIPLIST的指针
    unsigned int sz; // ZIPLIST的字节大小长度
    unsigned int count : 16; // ZIPLIST的entry个数
    unsigned int encoding : 2; // RAW==1 or LZF==2，编码方式，RAW=原始，LZF=压缩
    unsigned int container : 2; // NONE==1 or ZIPLIST==2
    unsigned int recompress : 1; // 前驱节点是否压缩？
    unsigned int attempted_compress : 1; // 无法压缩标识，即使指定了LZF也压缩不了
    unsigned int extra : 10; // 扩展空间
} quicklistNode;
```
快速列表示意：
![快速列表示意](@attachment/Clipboard_2021-05-25-00-44-18.png)
> 1. 直接就是一个双向链表，进行插入或删除操作时非常方便，虽然复杂度为O(n)，但是**不需要内存的复制，提高了效率**，而且访问两端元素复杂度为O(1)。
> 2. 每个entry继承了ZIPLIST的优点，顺序存储，内存连续，利用二分查找，对于ZIPLIST的**查找效率较高**。
> 3. 因为每个ZIPLIST都是小节点，默认不超过8kb，所以发生**连锁更新的情况也会比较小**。

### 集合对象（Set）
集合对象可支持的存储格式有2种：
> - OBJ_ENCODING_INTSET：当集合中的值全是整数，或者对象数量不超过512个时，采用整数集合(IntSet)。
> - OBJ_ENCODING_HT：不满足上述任一条件后，采用字典(HashTable)。

#### OBJ_ENCODING_INTSET
整数集合的结构定义如下：
```c
typedef struct intset {
    uint32_t encoding; // 整数的存储编码形式，INTSET_ENC_INT[16|32|64]，决定了编码后的字节大小
    uint32_t length; // 集合大小
    int8_t contents[]; // 元素数组
} intset;
```
整数集合示意：
![整数集合示意](@attachment/Clipboard_2021-05-25-20-54-12.png)
> 1. 采用了**数组作为存储结构，有序。使用二分查找**快速定位元素。
> 2. 不同编码会影响集合的空间申请大小，当整数由**小编码转大编码时，将数组扩容为大编码格式，需要扩容内存空间，反之则不会**。道理同ZIPLIST的扩容，为了防止抖动。

#### OBJ_ENCODING_HT
HT即HashTable，也就是字典。
字典的结构定义如下：
```c
typedef struct dict {
    dictType *type; // 类型特定函数
    void *privdata; // 指向私有数据的指针
    dictht ht[2]; // 2个Hash表，在进行扩容resize的时候，会用到第2个Hash表，保证读写不受影响。
    long rehashidx; // rehash目前的进度，没有发生rehash时为-1
    int16_t pauserehash; // 如果大于0表示rehashing暂停，如果小于0表示编码有错误
} dict;

typedef struct dictType {
    uint64_t (*hashFunction)(const void *key); // 计算Hash值
    void *(*keyDup)(void *privdata, const void *key); // 复制键
    void *(*valDup)(void *privdata, const void *obj); // 复制值
    int (*keyCompare)(void *privdata, const void *key1, const void *key2); // 对比键
    void (*keyDestructor)(void *privdata, void *key); // 销毁键
    void (*valDestructor)(void *privdata, void *obj); // 销毁值
    int (*expandAllowed)(size_t moreMem, double usedRatio); // 是否需要扩展空间
} dictType;

typedef struct dictht {
    dictEntry **table; // Hash表数组
    unsigned long size; // Hash表大小，数组总长度
    unsigned long sizemask; // Hash表长度size-1，用作索引
    unsigned long used; // 已使用的数组空间长度
} dictht;

typedef struct dictEntry {
    void *key; // 指向key的指针
    union {
        void *val; // 指向value的指针，非数值类型，指向另外的地址空间
        uint64_t u64; // 整数编码格式
        int64_t s64; // 整数编码格式
        double d; // 数值类型直接存这，不需要额外空间
    } v;
    struct dictEntry *next; // 指向下个entry节点
} dictEntry;
``` 
字典示意：
![字典示意](@attachment/Clipboard_2021-05-25-21-38-25.png)
> 1. **负载因子 = 哈希表当前保存节点数 / 哈希表大小**。当没有执行BGSAVE或BGREWRITEAOF时，负载因子大于等于1时就会发送扩展操作。否则当负载因子大于等于5时才会发生扩展操作。当负载因子**小于0.1时会发生收缩**操作。
> 2. 采用2张Hash表，渐进式扩展，不会明显降低当前hash表的访问效率。

### 有序集合对象（ZSet）
有序集合对象可支持的存储格式有2种：
> - OBJ_ENCODING_ZIPLIST：压缩列表，同上所述。
> - OBJ_ENCODING_SKIPLIST：跳跃表，类似于B*树，底层是一个双向链表，上层单向链表。

#### OBJ_ENCODING_SKIPLIST
跳跃表的结构定义如下：
```c
typedef struct zskiplist {
    struct zskiplistNode *header, *tail; // 跳跃表 头节点指针 尾节点指针
    unsigned long length; // 跳跃表的长度
    int level; // 跳跃表层数
} zskiplist;

typedef struct zskiplistNode {
    sds ele; // 直接存储字符串值的数据
    double score; // 存储排序的分值
    struct zskiplistNode *backward; // 后退指针，只能指向当前节点最底层的前一个节点，头节点和第一个节点—backward指向NULL，从后向前遍历跳跃表时使用。
    struct zskiplistLevel {
        struct zskiplistNode *forward; // 指向本层下一个节点，尾节点的forward指向NULL
        unsigned int span; // forward指向的节点与本节点之间的元素个数。span值越大，跳过的节点个数越多
    } level[]; // 为柔性数组。每个节点的数组长度不一样，在生成跳跃表节点时，随机生成一个1～64的值，值越大出现的概率越低。
} zskiplistNode;
```
跳跃表示意：
![跳跃表示意](@attachment/Clipboard_2021-05-26-22-46-18.png)
> 1. **允许score重复，由ele数据内容来唯一标识一份数据**。只有底层是双向链表，可以看做是level[0]的逆向链表。
> 2. 有序，且能快速查找列表中的元素，O(logN)的时间复杂度。
> 3. 插入和删除节点不会引起大范围的自平衡，只影响相邻节点。

### 哈希对象（Hash）
哈希对象可支持的存储格式有2种：
> - OBJ_ENCODING_ZIPLIST：压缩列表，同上所述。
> - OBJ_ENCODING_HT：字典，同上所述。

### 流对象（Stream）
流对象只有一种存储编码格式：
> - OBJ_ENCODING_STREAM：消息流，配合rax基数树（前缀树）一起使用

#### OBJ_ENCODING_STREAM
```c
typedef struct stream {
    rax *rax; // 存储生产者生产的具体消息，以消息ID为key，内容存储在rax中，rax中可以存储多个消息。
    uint64_t length; // 当前stream中的消息个数
    streamID last_id;  // 当前stream最后插入的消息ID，stream为空时等于0
    rax *cgroups; // 存储了当前stream的消费者组，rax中：name -> streamCG
} stream;

typedef struct streamID { // 消息ID（感觉有kafka那味了）
    uint64_t ms; // 以unix time的ms单位的时间戳
    uint64_t seq; // 序号
} streamID;

typedef struct rax {
    raxNode *head; // 指向头节点的指针
    uint64_t numele; // 元素个数（即key的个数）
    uint64_t numnodes; // 节点个数
} rax;

typedef struct raxNode {
    uint32_t iskey:1; // 节点是否包含了key，0=没有，1=包含key
    uint32_t isnull:1; // 节点的value是否为null，云数据只有key，没有value。
    uint32_t iscompr:1; // 节点是否被压缩，多个相同前缀合并
    uint32_t size:29; // 节点的大小，即存储的字符数

    /* iscompr=0：非压缩模式下，数据格式是：[header strlen=0][abc][a-ptr][b-ptr][c-ptr](value-ptr?)，
     *   有size个字符，紧跟着是size个指针，指向每个字符对应的下一个节点。size个字符之间互相没有路径联系。
     * iscompr=1：压缩模式下，数据格式是：[header strlen=3][xyz][z-ptr](value-ptr?)，
     *   只有一个指针，指向下一个节点。size个字符是压缩字符片段
     */
    unsigned char data[];
} raxNode;
```
前缀树示意：
![前缀树示意](@attachment/Clipboard_2021-05-27-00-04-16.png)
> 从左向右依次为，未压缩的rax树，压缩后的rax树，新增节点后的rax树变化。每个节点内容只存储压缩后的字符，当树出现分支时，由标记iscompr来标识当前data内容为非压缩节点，每个字符都有子节点（所以这就是为什么加入first之后，会有一个(fo) [o]出现，不合并到前面的分支或后面的分支，如果中间有超过一个的字符出现，那么也是会压缩的）。插入元素会导致节点分裂，同理，删除元素时又会再次将相同前缀的节点压缩。

所以上述最后一张图的内存结构如下所示：
![前缀树内存结构](@attachment/Clipboard_2021-05-27-00-36-32.png)
> 1. **没有被压缩的节点，每个值都有一个子节点**。
> 2. 普通节点的iskey=0，isnull=1。表明当前节点不是实际值，只有最后一个节点才是，它的iskey=1，isnull=0。
> 3. 被压缩的节点只有最后一个值作为指向子节点的指针。

Stream示意：
![Stream示意](@attachment/Clipboard_2021-05-27-00-52-39.png)
> 1. rax最后一个值另外充当了listpack的指针，标识当前key指向的消息流的实际内容。

## 参考资料
Redis 6.2.3源码
[redis 系列，要懂redis，首先得看懂sds（全网最细节的sds讲解）](https://blog.csdn.net/qq_33361976/article/details/109014012?utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7Edefault-8.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7Edefault-8.control)
[redis源码分析-intset(整型集合)](https://blog.csdn.net/Mijar2016/article/details/52197963)
[Redis 压缩列表(ziplist)和快速列表(quicklist)](https://blog.csdn.net/qq_20853741/article/details/111946054)
[Redis五种数据类型详解](https://www.cnblogs.com/qmillet/p/12494469.html)
[Redis数据结构——快速列表(quicklist)](https://www.cnblogs.com/hunternet/p/12624691.html)
[Redis数据结构——dict（字典）](https://blog.csdn.net/cxc576502021/article/details/82974940)
[Redis radix tree源码解析](https://zhuanlan.zhihu.com/p/64307643)
[带你读《Redis 5设计与源码分析》之三：跳　跃　表](https://developer.aliyun.com/article/727242)
[Redis5设计与源码分析 (第8章　Stream)](https://www.cnblogs.com/coloz/p/13812840.html)
