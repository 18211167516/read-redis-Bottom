## Redis服务器事件解读
1. 文件事件
>1、Redis服务器通过套接字与客户端（或者其他Redis服务器）进行连接，文件时间就是服务器对套接字操作的抽象  
>2、基于Reactor模式开发了自己的网络事件处理器，文件事件处理器  
>>2.1、使用I/O多路复用程序来同时监听多个套接字，并且关联不同的事件处理器  
>>2.2、当被监听的套接字准备好执行链接应答（accept）、读取（read）、写入（write）、关闭（close）等操作  
>>2.3、套接字->I/O多路复用程序（放入队列，依次出队列）->文件事件分派器（根据套接字的事件类型，分配）->事件处理器（映射关系)  
2. I/O多路复用
>1、select，epoll，evport，kqueue，程序在编译的时候会自动选择性能最高的函数库  
>2、事件类型：AE_READABLE（write 可读，close，accept）优先处理,AE_WRITABLE（read 可写），先读套接字，后写套接字  
>3、Api：  
>>3.1、aeCreataFileEvent 参数：套接字描述符、事件类型、事件处理器；功能： 加入监听范围，并对事件和事件处理器进行关联   
>>3.2、aeDeleteFileEvent 参数：套接字描述符、事件类型；功能：取消监听范围，并对事件和事件处理器取消关联  
>>3.3、aeGetFileEvents   参数：套接字描述符；功能：返回事件类型；预期：AE_NONE(没有事件监听)、AE_READABLE(读事件)、AE_WRITABLE(写事件)、AE_READABLE|AE_WRITABLE（读写同时）  
>>3.4、aeWait            参数：套接字描述符、事件类型、毫秒数；功能：在给定的时间内阻塞并等待给定事件产生或者超时返回  
>>3.5、aeApiPoll         参数：timeval结构体；功能：在置顶的时间内阻塞等待所有被aeCreateFileEvent函数设为监听状态的套接字产生事件，至少一个事件产生或者超时返回  
>>3.6、aeProcessEvents   参数：无；功能：事件分配器，调用aeApiPoll等待事件产生，并且遍历所有已产生的事件，调用对应的事件处理器  
>>3.7、aeGetApiName      参数：无；功能：返回I/O多路复用底层函数库名称；预期：select，epoll
3. 事件处理器
>1、连接应答处理器，当Redis服务器初始化时，使用aeCreateFileEvent注册关联AE_READABLE事件，当客户端监听socket，产生事件，引发处理器处理，并且创建一个新的客户端套接字，并且将该套接字AE_READABLE与命令请求处理器关联    
>2、命令处理器，当客户端通过连接应答处理器成功连接到服务器，关联事件AE_READABLE,当客户端发送请求时，产生事件，引发处理器处理  （一直关联）
>3、命令回复处理器，当服务器有回复需要传给客户端的时候，关联事件AE_WRITABLE,当客户端准备好接受服务器传回的命令回复时，产生事件，引发命令回复处理器执行（解除关联）
4. 时间事件
>Redis服务器中的一些操作（serverCron）需要在给定的时间点执行，而实践事件就是服务器对这类定时操作的抽象  
>分类：定时事件、周期事件  
>时间时间属性组成：id：全局唯一ID，从小到大；when：毫秒级UNIX时间戳,事件到达时间；timeProc:时间事件处理器，返回AE_NOMORE(定时事件，处理后删除)，返回非AE_NOMORE的整数值（周期事件，更新when时间，例返回30就将when更新为当前时间+30毫秒）
5. 时间事件Api
>1、aeCreateTimeEvent 参数：毫秒数、时间事件处理器；功能：添加一个新的时间事件  
>2、aeDeleteTimeEvent 参数：时间事件ID；功能：删除对应的时间事件  
>3、aeSearchNearestTimer 参数：无；功能：返回到达时间距离当前时间最接近的一个时间事件  
>4、processTimeEvents 参数：无；功能：时间事件的执行器，遍历所有到达的时间事件，并调用处理器  
6. 时间事件应用实例：serverCron
>1、更新服务器各类统计信息  
>2、过期策略（定期删除）  
>3、关闭和清理连接失败的客户端  
>4、持久化（AOF、RDB）  
>5、主服务器会定期同步从服务器  
>6、如果处于集群模式，对集群定期同步和连接测试  
>7、配置redis.conf里hz选项可以修改每秒执行次数  
7. 时间调度和执行
>1、先aeSearchNearestTimer获取最近的一条时间事件  
>2、判断距离最近的时间事件还剩多少毫秒remaind_ms  
>3、创建timeval结构体（传入remaind_ms）
>4、调用aeApiPoll（timeval）  
>5、processFileEvents()//处理所以已产生的文件事件  
>6、processTimeEvents()//处理所以已产生的时间事件
