## 数据库



### 数据库的结构

```c
typedef struct redisDb {
    dict *dict;                 /* The keyspace for this DB */
    dict *expires;              /* Timeout of keys with a timeout set */
    dict *blocking_keys;        /* Keys with clients waiting for data (BLPOP)*/
    dict *ready_keys;           /* Blocked keys that received a PUSH */
    dict *watched_keys;         /* WATCHED keys for MULTI/EXEC CAS */
    int id;                     /* Database ID */
    long long avg_ttl;          /* Average TTL, just for stats */
    unsigned long expires_cursor; /* Cursor of the active expire cycle. */
    list *defrag_later;         /* List of key names to attempt to defrag one by one, gradually. */
} redisDb;

struct redisServer {
		//...
  	redisDb *db;
  	int dbnum;                      /* Total number of configured DBs */
  	//...
}
```



​		对数据库数据的操作，如增删改查其本质就是对RedisDb结构中dict字典的操作以及底层数据结构的操作。具体可看底层数据结构的实现，在这里不做过多描述。



### 设置过期时间

redis有四个不同的过期时间命令：

- EXPIRE <key> <ttl> 命令用于将key的生存时间设置为ttl秒
- PEXPIRE <key> <ttl> 命令用于将key的生存时间设置为ttl豪秒
- EXPIREAT <key> <timestamp> 命令用于将key的生存时间设置为timestamp所指定的秒数时间戳
- EXPIRE <key> <timestamp> 命令用于将key的生存时间设置为timestamp所指定的豪秒数时间戳

虽然有多种不同单位不同形式的设置命令，但是在设置时，命令会转换成PEXPIREAT来执行，最终的执行效果和执行PEXPIREAT命令时一样的。



### 保存过期时间

redisDb结构中的expires字典保存了数据库所有的键的过期时间。

- expires字典的key是一个指针，这个指针指向dict字典的某个key对象（也就是某个要设置的key对象）
- expires字典的value是一个long long类型的整数，这个整数保存了key的过期时间（一个毫秒精度的UNIX的时间戳）



### 过期键删除策略

过期键有三种不同的删除策略

- 定时删除：在设置键的过期时间的同时，创建一个定时器，让定时器在键的过期时间来临，立即执行对键的删除操作。
- 惰性删除：放任键过期不管，但是每次从dict字典中获取键时，都去检查取得的键是否过期，如果过期的话，就删除该键；如果没有过期，就返回该键。
- 定期删除：每隔一段时间，程序就对数据库进行一次检查，删除里面的过期键。

Redis key过期的方式有三种：

- 惰性删除：当读/写一个已经过期的key时，会触发惰性删除策略，直接删除掉这个过期key（无法保证冷数据被及时删掉）
- 定期删除：Redis会定期主动淘汰一批已过期的key（随机抽取一批key检查）
- 内存淘汰机制：当前已用内存超过maxmemory限定时，触发主动清理策略

```
当内存达到限制时，Redis 具体的回收策略是通过 maxmemory-policy 配置项配置的。默认使用的是volatile-lru。

以下的策略都是可用的：

noenviction：不清除数据，只是返回错误，这样会导致浪费掉更多的内存，对大多数写命令（DEL 命令和其他的少数命令例外）
allkeys-lru：从所有的数据集（server.db[i].dict）中挑选最近最少使用的数据淘汰，以供新数据使用
volatile-lru：从已设置过期时间的数据集（server.db[i].expires）中挑选最近最少使用的数据淘汰，以供新数据使用
allkeys-random：从所有数据集（server.db[i].dict）中任意选择数据淘汰，以供新数据使用
volatile-random：从已设置过期时间的数据集（server.db[i].expires）中任意选择数据淘汰，以供新数据使用
volatile-ttl：从已设置过期时间的数据集（server.db[i].expires）中挑选将要过期的数据淘汰，以供新数据使用
```

​	

