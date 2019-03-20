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
REDIS_ENCODING_INTSET      | 整数集合
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

