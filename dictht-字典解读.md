## 前言：redis数据结构字典解读
1. 字典所使用哈希表结构定义：
```
tyoedef struct dictht{
    //哈希表数组
    dicEntry **table;
    //哈希表大小
    unsigned long size;
    //哈希表大小掩码，用于计算索引值，总是size-1
    unsigned long sizemask;
    //已存在的节点数量
    unsigned long used
}
```
2. 哈希表节点结构定义：
```
typedef struct dictEntry{
    //键
    void * key;
    //值
    union{
      void *val;//指针
      uint64_tu64;//unit64_t整数
      int64_ts64;//int64_t整数
    } v;
    //指向另一个哈希表节点指针（解决键冲突问题，多个哈希值相同的键值对连接在一起）
    struct dictEntry *next;
}
```
3. redis字典结构定义：
```
typedef struct dict{
    //；类型特定函数
    dictType *type;
    //私有数据
    void *privdata;
    //哈希表
    dictht ht[2];//通常使用ht[0] ht[1]在哈希表rehash时使用
    //当rehash不在进行时，值为-1
    int trehashidx;
} dict;
```
4. 特性
 * 4.1 采用 Murmurhash算法
 * 4.2 解决 键冲突哈希表节点next属性应用
 * 4.3 rehash（重新散列）扩展或者收缩哈希表
 * 4.4 渐进式rehash
5. 渐进式rehash步骤
```
1. 为ht[1]分配空间，让字典同时拥有ht[0],ht[1]
2. 在字典中维持一个rehashidx索引计数器变量，设为0
3. 在rehash期间，除了正常操作，还会顺带将ht[0]在rehashidx上的所有键值对rehash到ht[1],rehash完成后，rehashidx+1
4. rehash全部完成后，rehashidx设为-1，表示rehash成功，并且将ht[1]设为ht[0]，然后为ht[1]分配一个空白哈希表

```
6. 应用
```
数据库、哈希键
```
