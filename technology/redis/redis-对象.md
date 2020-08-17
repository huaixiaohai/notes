## RedisObject



`Redis`主要包含五大数据数据类型：字符串对象，列表对象，集合对象，有序集合对象，哈希对象。

`Redis`并没有对这五种对象实现具体的结构定义，而是使用统一的数据结构**`RedisObject`**,通过宏定义区分五种对象。

```c
typedef struct redisObject {
    unsigned type:4;	// 区分五种对象的宏定义类型
    unsigned encoding:4;	// 底层编码实现的宏定义类型
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount; // 引用计数
    void *ptr;	// 数据指针
} robj;

/* lru字段：
				1. 当使用lru算法，这个字段用来记录对象的热度。对象被创建时会记录 lru 值。在被访问的时候也会更新 lru 的值，但不是获取系统当前的时间戳，而是设置为全局变量 server.lruclock 的值。
				2. 当使用lfu算法，其被分为两部分：
    						高 16 位用来记录访问时间（单位为分钟，ldt，last decrement time）
    						低 8 位用来记录访问频率，简称 counter（logc，logistic counter）
								counter 是用基于概率的对数计数器实现的，8 位可以表示百万次的访问频率。
								对象被读写的时候，lfu 的值会被更新。

*/

/* The actual Redis Object */
#define OBJ_STRING 0    /* String object. */
#define OBJ_LIST 1      /* List object. */
#define OBJ_SET 2       /* Set object. */
#define OBJ_ZSET 3      /* Sorted set object. */
#define OBJ_HASH 4      /* Hash object. */


/* 底层数据结构的类型编码 */
#define OBJ_ENCODING_RAW 0     /* Raw representation 简单动态字符串*/
#define OBJ_ENCODING_INT 1     /* Encoded as integer */
#define OBJ_ENCODING_HT 2      /* Encoded as hash table */
#define OBJ_ENCODING_ZIPMAP 3  /* Encoded as zipmap */
#define OBJ_ENCODING_LINKEDLIST 4 /* No longer used: old list encoding. */
#define OBJ_ENCODING_ZIPLIST 5 /* Encoded as ziplist */
#define OBJ_ENCODING_INTSET 6  /* Encoded as intset */
#define OBJ_ENCODING_SKIPLIST 7  /* Encoded as skiplist */
#define OBJ_ENCODING_EMBSTR 8  /* Embedded sds string encoding （embstr编码的简单动态字符串）*/
#define OBJ_ENCODING_QUICKLIST 9 /* Encoded as linked list of ziplists */
#define OBJ_ENCODING_STREAM 10 /* Encoded as a radix tree of listpacks */
```



不同对象对应的底层编码方式

| 对象类型   | 对象底层编码方式        |
| ---------- | ----------------------- |
| OBJ_STRING | OBJ_ENCODING_INT        |
| OBJ_STRING | OBJ_ENCODING_RAW        |
| OBJ_STRING | OBJ_ENCODING_EMBSTR     |
| OBJ_LIST   | OBJ_ENCODING_ZIPLIST    |
| OBJ_LIST   | OBJ_ENCODING_LINKEDLIST |
| OBJ_HASH   | OBJ_ENCODING_ZIPLIST    |
| OBJ_HASH   | OBJ_ENCODING_HT         |
| OBJ_SET    | OBJ_ENCODING_INTSET     |
| OBJ_SET    | OBJ_ENCODING_HT         |
| OBJ_ZSET   | OBJ_ENCODING_SKIPLIST   |
| OBJ_ZSET   | OBJ_ENCODING_SKIPLIST   |

