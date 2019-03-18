## 前言：关于redis 链表数据结构读后解读

1. 链表节点listNode定义：
```
typedef struct listNode{
    struct listNode * prev;//前置节点
    struct listNode * next;//后置节点
    void * value;//节点的值
}listNode
```
2. 链表list结构体定义：
```
typedef struct list{
    //表头节点
    listNode * head;
    //表尾节点
    listNode * tail;
    //包含的节点数量
    unsigned long len;
    //节点值复制函数
    void *(*dup)(void *ptr);
    //节点值释放函数
    void (*free)(void *ptr);
    //节点值对比函数
    int (*match)(void *ptr,void *key);
} list
```
3. redis链表实现的特性：
 * 3.1 双端：每个链表节点都带有prev和next指针，获取前一，后一节点复杂度都是O(1);
 * 3.2 无环：表头节点的prev和表尾节点的next节点都指向NULL，对链表的访问已NULL结束
 * 3.3 获取表头和表尾节点的复杂度都胃O(1);
 * 3.4 使用list结构中len属性来对list持有的节点进行计数，获取列表长度复杂度为O(1);
 * 3.5 多态：链表节点通过void*来保存节点值，可以保存不同类型的值
4. 应用：
```
列表键、发布和订阅、监视器、慢查询等
```
