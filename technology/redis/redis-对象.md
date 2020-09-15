## 对象



### 对象的类型与编码

`Redis`主要包含五大数据数据类型：字符串对象，列表对象，集合对象，有序集合对象，哈希对象。

`Redis`并没有对这五种对象实现具体的结构定义，而是使用统一的数据结构**`RedisObject`**，结合底层数据结构创建的一个对象系统。

```c
typedef struct redisObject {
    unsigned type:4;	// 区分五种对象的宏定义类型
    unsigned encoding:4;	// 底层编码实现的宏定义类型，（使用c语言的位域，节省内存）
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

redis之所以设计这样的对象系统，主要的好处：

- 可以自由的改进底层数据结构的编码实现，这样一旦开发出更优秀的内部底层编码，无需修改对外的对象和命令。
- 在不同的使用场景下，对对象设置多种不用的数据结构实现，以优化对象在不同场景下的使用效率。

### 内存回收

​		因为c语言并不具备自动内存回收功能，所以redis在自己的对象系统中构建了一个引用计数技术实现的内存回收机制，通过这一机制，程序可以通过跟踪对象的引用计数信息，在适当的时候自动释放对象并进行内存回收。

对象的引用计数信息会随着对象的使用状态而不断变化：

- 在创建一个新对象时，引用计数值会被增一；
- 在对象被一个新程序使用时，引用计数值会被增一；
- 当一个对象不再被一个程序使用时，它的引用计数值会这减一；
- 当对象的引用计数值变为0时，该对象就会被释放；

### 对象共享

​		引用计数除了用于内存回收机制外，引用计数还带有对象共享的作用。Redis在在初始化服务器时，创建一万个字符串对象，这些对象包含了从0～9999的所有的整数值，当服务器需要用到值为0～999的字符串对象时，服务器就会使用共享对象，而不是创建新对象。

### 对象的空转时长

​		RedisObject结构中包含一个lru属性，该属性记录了对象最后一次被命令程序访问的时间（除object idletime命令）。

​		命令object idletime命令可以打印出给定键的空转时长，空转时长就是当前时间减去键的值对象的lru时间。

​		空转时长另外的一项作用：如果服务器打开了maxmemory选项，并且服务器用于回收内存的算法为volatile-lru或者allkeys-lru，那么当服务器占用的内存数超过了maxmemory选项设置的上限值时，空转时长较高的那部分键会被优先释放，从而回收内存。



​		

