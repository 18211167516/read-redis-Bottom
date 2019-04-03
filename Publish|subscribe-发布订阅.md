## 前言：Redis的发布与订阅功解读 
1. 频道的订阅与退订
> 1、当一个客户端执行subscribe命令订阅某个或某些频道的时候，就会建立订阅关系，Redis将所有频道的订阅关系就保存在服务器状态pubsub_channels字典里  
> 2、当客户端订阅某频道时，服务会为频道和客户端在pubsub_channels里做关联，存在的话就直接添加到链表尾部；否则，先新建一个键，创建一个空链表，在将客户端添加到链表  
> 3、当客户端执行unsubscribe命令时，服务器将从pubsub_channels里解除关联

2. 模式的订阅与退订
> 1、服务器将所有模式的订阅关系都保存在服务器状态的pubsub_patterns属性里  
> 2、当客户端执行psubscribe订阅某个模式的时候，服务器会新建一个pubsubPattern结构，将结构添加到pubsub_patterns链表尾部  
> 3、当客户端执行punsubscribe退订某个模式的时候，服务器会接触关联  
```
typedef struct pubsubPattern{
    //订阅模式的客户端
    redisClient *client;
    //被订阅的模式
    robj *pattern;
}
```
3. 发送消息  
> 1、当客户端执行publish <channel> <message>，服务器将message发送给channel的所有订阅者，如果有一个或者多个模式pattern与频道channel相匹配，那么message也会发送给pattern模式的订阅者  
> 2、遍历pubsub_channels字典里 channel对应的链表发送消息，遍历所有pubsub_patterns链表查找订阅模式是否和channel相匹配，如果找到匹配，遍历发送  
4. 查看订阅信息 
> 1、客户端可通过 pubsub 查看频道或者模式的相关信息  
> 2、pubsub channels[pattern] 遍历pubsub_channels  （例子：pubsub channels "new.[is]*"）  
> 3、pubsub numsub[channel-1 channel-2]返回对应频道的订阅者数量，通过在pubsub_channels字典中找到频道对应的订阅者链表，然后返回链表的长度实现  
> 4、pubsub numpat 返回服务器当前被订阅模式的数量，返回pubsub_patterns链表的长度

```
struct redisServer{
  //保存所有频道的订阅关系，键是某个被订阅的频道，而键的值则是一个链表（记录所有订阅这个频道的客户端）
  dict *pubsub_channels;
  //保存所有模式的订阅关系
  list *pubsub_patterns;
}
```
