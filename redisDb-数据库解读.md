## 前言：redis服务器的数据库解读
1. redisServer结构定义
```
struct redisServer{
    //一个数组，保存着服务器中所有数据库
    redisDb *db;
    //数据库数量（由配置的database选项决定，默认16）
    int dbnum;
    //记录保存条件的数组
    struct saveparam *saveparamsl;
    //修改计数器
    long long dirty;
    //上一次执行保存的时间
    time_t lastsave;
    //AOF缓存区
    sds aof_buf;
}
//RDB 持久化条件数组结构体
struct saveparam{
     //秒数
     time_t seconds;
     //修改数
     int changes;
}
```
2. redisClient结构定义
```
typedef struct redisClient{
  //记录客户端当前正在使用的数据库
  redisDb *db;//指向redisServer.db数组中其中一个元素

} redisClient
```
3. 切换数据库
```
1. 默认操作数据库0；
2. select 2；//切换到2号数据库
```
4. 数据库键空间
```
typedef struct redisDb{
  //数据库键空间，保存着数据库所有键值对
  dict *dict
  //过期字典，保存着键的过期时间
  dict *expires;
} redisDb
```
5. 添加，删除，更新，对键取值，其他针对数据库本身的redis操作都是对键空间进行处理
6. 设置键的生存时间或过期时间
```
1. Expire Pexpire命令设置以秒为单位的生存时间
2. Expireat Pexireat命令设置unix时间戳的生存时间
```
7. 移除键的过期时间
```
1. Persist (在过期字典中查找给定的键，并解除键和值的关联)
```
8. 计算并返回剩余时间
```
1. TTL 返回剩余秒数  
2. PTTL返回剩余毫秒时间
```
9. 过期键的判定
```
1. 检测给定的键是否存在过期字典，如果存在，返回过期时间
2. 检测当前时间戳是否大于键的过期时间，如果是，已过期，否则，未过期
```
10. Redis的过期键删除策略
```
1. 定期删除 （定时100ms，随机检测）
2. 惰性删除（访问的时候，判断）
```
11. RDB持久化
```
1. 执行save命令或者bgsave，生成新的RDB文件时，已过期的键不会保存
2. 载入RDB文件的时候，主服务器（过期键会被忽略），从服务器（全量载入，主从同步时会被清空）
```
12. AOF持久化
```
执行bgrewriteaof
1. 写入的时候，如果键已经过期，但没有删除，当过期删除的时候，会追加一条del命令
2. 重写的时候，过滤已过期的键
```
13. 复制模式
```
1. 主服务器控制统一删除过期键，保证主从数据一致性
``
