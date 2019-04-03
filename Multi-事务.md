## 前言：Redis事务解读
1. 事务的实现
> 1、事务开始：multi命令标志事务的开始，将执行该命令的客户端flags打开REDIS_MULTI标识，切换到事务状态  
> 2、事务入队：当一个客户端切换到事务状态时，如果发送命令为（exec,discard,watch,multi）立即执行，否则将命令放入事务队列里，返回queued回复  
> 3、事务队列：已先进先出的方式保存入队的命令  
> 4、执行事务：发送exec命令时：服务器会遍历这个客户端的事务队列，执行保存在队列中的所有命令，最后将执行命令所得的结果全部返回客户端  
```
客户端状态
typedef struct redisClient{
    //事务状态
    multiState mstate;
}

typedef struct multiState{
    //事务队列 FIFO顺序
    multiCmd *commands;
    //已入队命令计数
    int count;
}

typedef struct multiCmd{
    robj *argv;//参数
    int arbc;//参数数量
    struct redisCommand *cmd;//命令指针
}
```
2. watch命令的实现
> 1、watch是一个乐观锁，可以在exec命令执行之前，监视任意数量的数据库键，在exec执行时，检测如果被监视的键至少有一个已经被修改过了，那么服务器将拒绝执行事务，返回事务失败  
> 2、每个Redis数据库保存了一个watched_keys字典，这个字典的键是某个被watch监视的数据库键，而字典的值则是一个链表，链表记录了所有监视相应数据键的客户端  
3. 监视机制的触发
> 1、所有对数据库进行修改的命令，在调用之后都会函数对watched_keys字典检查，如果有修改，就会将监视被修改键的客户端的REDIS_DIRTY_CAS标识打开，标识该客户端事务安全性已经被破坏 
> 2、服务器在接受到客户端exec命令时，服务器会根据这个客户端是否打开REDIS_DIRTY_CAS，如果打开不执行，否则执行
4. 事务的ACID性质
> 1、原子性（Atomicity）：所有操作要么执行要么都不执行（不支持回滚）  
> 2、一致性（Consistency）：执行事务前后数据库保持数据一致；入队错误，事务拒绝执行；执行错误：会继续执行剩余命令，已执行的命令不会被出错的命令影响；服务器停机：如果没有开始持久化，重启后数据空白，数据一致；如果开始持久化，重启后会恢复数据，总是保持数据一致  
> 3、隔离性（Lsolation）： 单线程执行事务，redis的事务已串行的方式运行具有隔离性    
> 4、耐久性（Durability）：取决于redis是否启用持久化（在AOF模式下 appendfsync = always）
