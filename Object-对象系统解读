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
3. type
```
类型常量      | 对象的名称   | TYPE命令返回
REDIS_STRING | 字符串对象   | string
REDIS_list   | 列表对象     | list
REDIS_HASH   | 哈希队列     | hash
REDIS_SET    | 集合对象     | set
REDIS_ZSET   | 有序集合对象 | zset
```
4. encoding
```
12313
```
5. 1232
header 1 | header 2
---|---
row 1 col 1 | row 1 col 2
row 2 col 1 | row 2 col 2
