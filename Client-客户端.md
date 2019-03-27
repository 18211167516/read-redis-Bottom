## 前言：Redis客户端解读
1. 使用I/O多路复用 ,Redis服务器使用单线程单进程的方式来处理命令请求，并与多个客户端进行网略通信
```
1、每个链接的客户端，服务器都会为这些客户建立相应的eredisClinet结构（状态），保存了当前客户端的状态信息以及执行相关功能需要的数据结构
2、服务器会在 redisServer {
      list *clients;//保存所有客户端状态
}
```
2. redisClient结构体定义
```
typedef struct redisClient{
    //套接字描述符
    int fd;
    //名字（默认指向NULL）
    robj *name;
    //标志
    int flags;
    //输入缓存区
    sds querybuf;
    //命令参数
    robj **argv；
    //参数个数
    int argc;
    //命令实现函数
    struct redisCommand *cmd;
    //输出缓存区（固定缓存区）
    char buf[REDIS_REPLY_CHUNK_BYTES]；//默认16kb 16*1024；
    //当前buf数组已使用字节数量
    int bufpos;
    //可变大小输出缓存区链表
    list *reply;
    //身份验证
    int authenticated;
    //创建时间（计算连接多少秒  ）
    time_t ctime;
    //最后一次互动时间(计算空转时间 idle)
    time_t lastinteraction;
    //输出缓存区达到软性限制的时间
    time_t obuf_soft_limit_reached_time;
}
```
3. 客户端属性
>1、fd（套接字描述符），-1：伪客户端（场景：载入AOF文件还原数据库状态，执行Lua脚本）；普通客户端大于1；client list(列出所有普通客户端)  
>2、name （名字），默认空；客户端使用client setname(设置名称)  
>3、flags (标志)，可以是单个标志，也可以是多个标志的二进制，每个标志使用一个常量标识  
>4、querybuf （输入缓存区），会动态的缩小或者扩大，但不可以超过1GB，否则直接关闭客户端  
>5、argv （命令参数）argv[0] 执行的命令，其他是参数项  
>6、argc （参数个数）  
>7、redisCommand (命令实现函数)  
>8、buf、bufpos（固定输出缓存区），主要用来回复长度较短的回复  
>9、reply（可变大小输出缓存区）。当buf数组的空间已经用完，或者长度超过16kb  
>10、authenticated (身份验证) 0 未认证（只允许auth命令执行） 1 已通过（如果服务器开启身份验证的话）  
>11、ctime （客户端创建时间），计算客户端连接了多少秒 client list命令age  
>12、lastinteraction（最后一次互动时间），计算客户端空转时间 （idle），client list命令idel  
>13、obuf_soft_limit_reached_time（输出缓存区达到软性限制的时间）  
4. flags标志的常见常量
> 1、REDIS_MASTER (主从复制操作时)，代表主服务器  
> 2、REDIS_SLAVE (主从复制操作时)，代表从服务器  
> 3、REDIS_PRE_PSYNC 标识客户端版本低于Redis2.8的从服务器，主服务器不可使用，必须REDIS_SLAVE标志打开  
> 4、REDIS_LUACLIENT (专门处理Lua脚本的伪客户端)  
> 5、REDIS_MONITOR （表示客户端正在执行MONITOR命令）  
> 6、REDIS_UNIX_SCOKET （表示服务器使用UNIX套接字来链接客户端）  
> 7、REDIS_BLOCKED (表示客户端正在被BRPOP、BLPOP命令阻塞)  
> 8、REDIS_UNBLOCKED (表示已经不阻塞了)  
> 9、REDIS_MULTI (表示正在执行事务)  
>10、REDIS_DIRTY_CAS（表示事务使用watch命令监视的数据库键已修改）  事务失败
>11、REDIS_DIRTY_EXEC (表示事务在命令入队时出现错误) 事务失败  
>12、REDIS_CLOSE_ASAP （表示客户端输出缓存区大小超过了服务器允许的范围），服务器会在下一次执行serverCron时关闭该客户端  
>13、REDIS_CLOSE_AFTER_REPLY (表示有用户对这个客户端执行了client kill)，关闭客户端  
>14、REDIS_ASKING （表示客户端向集群节点发送了ASKING命令）  
>15、REDIS_FORCE_AOF (表示强制服务器将当前执行的命令写入AOF)  执行pubsub命令会出现此标志
>16、REDIS_FORCE_REPL (强制主服务器将当前执行的命令复制给所有从服务器) 执行script load 会出现此标志  
5. 客户端创建
```
1、创建普通客户端，服务器会调用连接事件处理器，为客户端创建相应的状态。并添加到服务器状态结构clients链表尾部
2、Lua脚本创建的伪客户端（服务器关闭才会关闭）
3、AOF文件载入创建伪客户端，并在载入成功后，关闭
```
6. 客户端关闭（常见场景）
>1、客户端进程退出或者杀死  
>2、客户端向服务器发送了不符合协议格式的命令请求  
>3、使用client kill  
>4、服务器设置了timeout配置项，客户端空转时间超过这个值被关闭，特殊情况（REDIS_MASTER|REDIS_SLAVE|REDIS_BLOCKED或者执行发布订阅命令不会关闭）  
>5、命令请求超过了输入缓存区限制大小（默认1GB）  
>6、如果发送客户端的命令回复大小超过了输出缓存区的限制大小  
>> 6.1、硬性限制（hard limit），超过这个大小，直接关闭  
>> 6.2、软性限制 （soft limit）,如果超过了这个大小，但没有超过硬性限制，将使用客户端状态obuf_soft_limit_reached_time记录起始时间，之后一直监视，如果持续超过软性限制，并且超过设定时长，关闭，如果在指定时间内，没有超过obuf_soft_limit_reached_time归零  
>> 6.3、使用client-output-bugger-limit <class> <hard limit> <soft limit> <soft srconds> 命令可以为普通客户端，从服务器客户端，执行发布于订阅客户端设置限制
