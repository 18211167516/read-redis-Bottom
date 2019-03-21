## 前言：redis对象系统解读
1. 对象
```
1. 字符串对象
2. 列表对象
3. 哈希对象
4. 集合对象
5. 有序集合对象
```
2. Object结构定义
```
typedef struct redisObject{
    //类型
    unsigned type:4;
    //编码指定底层用什么数据结构实现
    unsigned encoding:4;
    //指向底层实现数据结构的指针
    void *ptr;
    //引用计数  内存回收机制
    unsigned refcount;
    //空转时长 （当前时间减去最后一次访问时间）
    unsigned lru；22；
} robj
```
3. type (type key 命令)
```
类型常量      | 对象的名称   | TYPE命令返回
REDIS_STRING | 字符串对象   | string
REDIS_list   | 列表对象     | list
REDIS_HASH   | 哈希队列     | hash
REDIS_SET    | 集合对象     | set
REDIS_ZSET   | 有序集合对象 | zset
```
4. encoding 编码 （object encoding）
```
编码常量                     | 编码对应数据结构  
REDIS_ENCODING_INT          | long类型的整数   
REDIS_ENCODING_EMBSTR       | embstr编码的简单动态字符串    
REDIS_ENCODING_RAW          | 简单动态字符串     
REDIS_ENCODING_HT           | 字典     
REDIS_ENCODING_LINKEDLIST   | 双端链表 
REDIS_ENCODING_ZIPLIST      | 压缩列表
REDIS_ENCODING_INTSET       | 整数集合
REDIS_ENCODING_SKIPLIST     | 跳跃表和字典
```
5. 不同类型和编码的对象
```

类型         |     编码                     | 对象
REDIS_STRING | REDIS_ENCODING_INT          | 使用整数值实现的字符串对象 
REDIS_STRING | REDIS_ENCODING_EMBSTR       | 使用embstr编码的简单动态字符串实现的字符串对象    
REDIS_STRING | REDIS_ENCODING_RAW          | 使用简单动态字符串实现的字符串对象     
REDIS_LIST   | REDIS_ENCODING_ZIPLIST      | 使用压缩列表实现的列表对象     
REDIS_LIST   | REDIS_ENCODING_LINKEDLIST   | 使用双端链表实现的列表对象 
REDIS_HASH   | REDIS_ENCODING_ZIPLIST      | 使用压缩列表实现的哈希对象
REDIS_HASH   | REDIS_ENCODING_HT           | 使用字典实现的哈希对象
REDIS_SET    | REDIS_ENCODING_INTSET       | 使用整数集合实现的集合对象
REDIS_SET    | REDIS_ENCODING_HT           | 使用字典实现的集合对象
REDIS_ZSET   | REDIS_ENCODING_ZIPLIST      | 使用压缩列表实现的有序集合对象
REDIS_ZSET   | REDIS_ENCODING_SKIPLIST     | 使用跳跃表和字典实现的有序集合对象
```
6. 字符串对象
```
strlen  字符串长度
1. 保存整数值，并且可以用long类型来标识，那么编码为int，如果对象保存的值不在是整数值，那么字符串对象的编码将变为raw
2. 保存字符串值，并且字符串字节数大于32字节，那么编码设置为raw
3. 保存字符串值，并且字符串字节数小于32字节，那么编码设为embstr（只读），执行修改支后会变成一个raw编码的字符串对象
4. 在某些情况下，程序会将保存在字符串对象中的字符串转为浮点数，执行某些操作
```
7. embstr的优点
```
1. 通过调用1次内存分配函数来分配一块连续的空间空间中一次包含redisObject和sdshdr
2. 释放embstr编码的字符串对象只需要调用一次内存释放函数，而释放raw编码需要调用俩次内存释放函数
3. 连续内存访问快
```
8. 列表对象
```
1. 3.2之后使用quicklist结构作为列表键底层实现
```
9. 哈希对象
```
1. 编码可以是ziplist 或者hashtable（字典）
2. 当哈希对象保存的键值对的键和值的字符串长度都小于64字节，并且键值对数量小于512；使用ziplist编码（配置文件可修改）
3. 当上述条件不满足时，使用hashtable编码
```
10. 集合对象
```
1. 编码可以是intset或者hashtable
2. 当集合对象保存的所有元素都是整数值，并且元素数量不超过512；
3. 不满足上述条件的集合对象需要使用hashtable编码
```
11. 有序集合对象
```
1. 编码可以是ziplist或者skiplist（同时包含一个字典和跳跃表，性能更高）
2. 当有序集合保存的元素数量小于128，并且元素成员的长度都小于64字节；
3. 不满足上述条件的集合对象需要使用skiplist

```
12. 类型检查和命令对态
```
1. 2中命令，第一种：可以对任何类型的键执行，第二种：特定类型的命令（会判断redisObject的type）
2. 多态是根据值对象的编码来选择正确的命令实现代码来执行命令
```
13. 内存回收
```
1. 引用计数的内存回收机制，在适当的时候自动释放对象并进行内存回收
2. redisObject里refcount属性
3. 创建对象，引用计数初始化为1
4. 当对象被一个新程序使用时，引用计数+1
5. 当对象不在被一个程序使用时，它的引用计数值会被-1
6. 当对象的引用计数值变为0时，释放内存
```
14. 对象共享（只对整数值的字符串对象进行共享）（object refcount）
```
初始化服务器的时候，创建一w个字符串对象，包含0-9999
```
15. 对象的空转时长 (object idletime)
```
1. 由redisObject里lru属性（当前时间-最后一次访问时间）
2. 如果服务器打开了maxmemory选项时，如果内存达到上限，会优先释放空转时间较长的键
```

