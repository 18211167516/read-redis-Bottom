## 前言：Sentinel 是Redis的高可用性解决方案
1. 启动并且初始化sentinel 
>1、使用命令 redis-sentinel /path/to/your/sentinel.conf | redis-server /path/to/your/sentinel.conf --sentinel  
>2、初始化服务器  
>3、将普通redis服务器使用的代码替换成sentinel专用代码  
>4、初始化sentinel状态  
>5、根据给定的配置文件，初始化sentinel的监视主服务器列表  
>6、创建连向主服务器的网络连接  
2. 初始化服务器：
> 初始化一个普通的redis服务器，但是有很多功能禁用  
3. 使用sentinel专用代码：
> 将一部分普通redis服务器使用的代码替换成sentinel代码  
4. 初始化sentinel状态：
> 在应用了sentinel代码后；会初始化一个sentinelState结构  
```
struct sentinelState{
  uint64_t current_epoch;
  //保存了所以被这个sentinel监视的主服务器
  //字典的键是服务器的名字
  //字典的值是一个指向sentinelRedisInstance结构的指针
  dict *masters;
  //是否进入TILT模式
  int tilt;
  //目前正执行的脚本的数量
  int running_scripts；
  //TILT模式开始时间
  mstime_t tilt_start_time;
  //最后一次执行时间处理器的时间
  mstime_t previous_time;
  //所有需要执行的用户脚本
  list *scripts_queue;
} sentinel
```
5. 初始化sentinel状态里的masters属性：
> sentinelRedisInstance 定义  
```
typedef struct sentinelRedisInstance{
      //标识值，实例类型和实例状态
      int flags;
      //实例名称
      //主服务器的名字由sentinel自动设置
      //格式为ip：port 
      char *name;
      //实例的运行id
      char *runid；
      //配置纪元，用于实现故障转移
      unit64_t config_epoch;
      //实例的地址
      sentinelAddr *addr;
      //sentinel down-after-milliseconds设定的值
      //实例无响应多少毫秒之后才会判断为主管下线
      mstime_t down-after_period;
      //sentinel monitor <master_name> <IP> <port> <quorum> 
      //判断这个实例为客观下线所需的支持投票数量 
      int quorum;
      // sentinel parallel-syncs <master-name> <number>
      //执行故障转移操作时，可以同时对新的主服务器进行同步的从服务器数量
      int parallel_syncs;
      //刷新故障迁移状态的最大时限
      mstime_t failover_timeout;
} sentinelRedisInstance
```
```
typedef struct sentinelAddr{
    char *ip;
    int port;
} sentinelAddr
```
> 根据载入的sentinel配置文件加载
6. 创建连向主服务器的网络连接：
> 1、创建2个异步连接：一个是命令连接，专门向主服务器发送命令，并接受命令回复；一个是订阅连接（为了不丢失发布订阅的消息），专门订阅主服务器的_sentinel_:hello频道     
> 2、获取主服务器信息：sentinel默认每10秒一次，通过命令连接向主服务器发送INFO命令，并分析主服务器信息：run_id和role记录的服务器角色；slave开头的从服务器信息，自动化发现从服务器，并保存到slaves字典中  
> 3、获取从服务器信息：创建向从服务器的网络连接，每10秒一次发送info命令；获取信息，更新实例  
> 4、向主，从服务器发送命令：默认会已每2秒一次；向所有被监视的主从服务器发送：publish _sentinel_:hello "<s_ip>,<s_port>,<s_runid>,<s_epoch>,<m_name>,<m_ip>,<m_port>,<m_epoch>"  
> 5、接受来自主从服务器的频道信息：sentinel既通过命令连接向服务器的频道发送消息，又通过订阅频道接受消息  
> 6、多个sentinel监视同一个服务器，会同时收到其中一台发送给服务器的频道消息；同时提取消息，通过s_runid判断是否自己发送，如果不是就更新对应实例结构  
> 7、更新sentinels字典：保存同样监视这个主服务器的其他sentinel的资料  
> 8、当通过频道消息发现新的sentinel时：在sentinels字典中新增；创建一个连向新sentinel的命令连接，同时新sentinel也会创建连向这个的连接  
> 9、检测主观下线：默认每秒一次向所有创建了命令连接的实例（主服务器，从服务器，其他sentinel），发送ping 命令，判断是否在线；
> 10、根据配置文件选项 down-after-milliseconds的值，在这个值的范围外一直没有响应就会判断为主观下线 (包括主从服务器，其他sentinel)，多个sentinel该选项的值不同  
> 11、检测客观下线状态：当某个sentinel将一个主服务器判断为主观下线时，会向其他监视这个主服务器的sentinel进行询问确认，当确认的值达到quorum时，会判定为客观下线，并对服务器进行故障转移  
> 12、发送sentinel is-master-down-by-addr <ip> <port> <current_epoch> <runid>   
> 13、接受到命令时：返回 <down_state>(1下线 0 未下线) <leader_runid>（*或者局部领头sentinel 运行id） <leader_epoch> （局部领头sentinel配置纪元）  
> 14、根据其他sentinel的回复：统计其他sentinel同意下线的数量，当达到客观下线所需数量，会将主服务实例结构flags属性的SRI_O_DOWN标识打开，表示已下线；  
> 15、选举头领sentinel：当一个主服务被判定为客观下线时，监视这个主服务器的所有sentinel要协商选择头领，由头领对下线主服务器处理故障转移  

7. 选举sentinel头领规则：
> 1、每一个都有机会  
> 2、无论选举成功，所有sentinel纪元配置+1  
> 3、在一个配置纪元里（也就是选举未完成时），所有sentinel都有一次投票权  
> 4、每个发现主服务器进入客观下线的sentinel会要求其他sentinel将自己设置为头领  
> 5、当一个源sentinel像目标sentinel发送 sentinel is-master-down-by-addr 里 runid不是*而是源sentinel的运行ID，要求目标sentinel将源sentinel设为头领  
> 6、先到先得  
> 7、目标sentinel回复，leader_runid （领头运行id）leader_epoch（配置纪元）  
> 8、源sentinel接受到回复之后，会检查leader_epoch是否和自己的配置纪元相同，如果相同，继续取出leader_runid，如果值和当前运行id一致  
> 9、如果有某个sentinel被半数以上的sentinel设置成了局部领头，那么sentinel会成功所有sentinel的领头  
> 10、每一个sentinel只会有一次选举局部领头的机会  
> 11、在一定的时间内，如果没有选举出领头sentinel，那么会在过一段时间再次选举，直到选举领头sentinel  

8. 故障转移
>1、在已下线的主服务器属下的所有从服务器里面，挑选出一个从服务器，将其转换为主服务器  
>2、让已下线主服务器属下所有的从服务器改为复制新的主服务器  
>3、将已下线主服务器设置为新的主服务器的从服务器，当这个旧的主服务器上线时，就会成功新的主服务器的从服务器  

9. 选出新的主服务器
> 1、挑选一个状态良好的，数据完整的从服务器，发送slaveof no one 命令，转为主服务器  
> 2、将所有从服务器保存到一个列表里  
> 3、删除列表中所有处于下线或者断线状态的从服务器，保证剩余的都是正常在线的  
> 4、删除列表中最近5秒内没有回复过头领sentinel的info命令的从服务器，保证剩余的从服务器都是最近成功进行通信的  
> 5、删除所有与已下线主服务器断开超过 down-after-milliseconds * 10毫秒的从服务器，保证剩余的从服务器的数据都是比较新的  
> 6、根据优先级，选出优先级最高的从服务器  
> 7、如果有相同的优先级，那么领头sentinel将选出偏移量最大的从服务器，如果偏移量也一样，就按运行id排序，选出最小的  
> 8、在发送slaveod no one命令后，领头sentinel会已每秒一次，向被升级的从服务器发送info命令，并观察命令回复中的角色（role）信息，当slave变为master时就知道升级成功  
10. 修改从服务器的复制目标 
> 1、新的主服务器升级成功  
> 2、向剩余的从服务器发送slaveof 命令复制新的主服务器
11. 将旧的主服务器变为从服务器
> 1、旧的主服务器上线后，sentinel会向他发送slaveof 命令，让他成为新的从服务器





