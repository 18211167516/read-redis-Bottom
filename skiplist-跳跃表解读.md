## 前言：redis数据结构跳跃表解读

1. 跳跃表节点定义
```
typedef struct zskiplistNode{
      //层
      struct zskiplistLevel{
          //前进指针
          struct zskiplistNode *forward;
          //跨度
          unsigned int span;//计算排位
      }
      //后退指针
      struct zskiplistNode *backward;//后退到前一位
      //分值
      double score;
      //成员对象 一个指针，字符串对象保存SDS值
      robj *obj;
} zskiplistNode
```
2. 跳跃表定义
```
typedef struct zskiplist{
    //表头节点和表尾节点
    struct skiplistNode *header,*tail;
    //表中节点的数量
    unsigned long length;
    //表中层数最大的节点的层数
    int level;
} zskiplist
```
3. 应用
```
有序集合
```
