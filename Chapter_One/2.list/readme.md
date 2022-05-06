# 链表
## 一、小结
1. redis中的链表数据结构总体由链表节点`listNode`、上层管理者`list`以及迭代器`listIter`构成。
2. `listNode`的定义中包含前置节点指针和后置节点指针，另外规定头结点的前置指针和尾节点的后置指针置空，故redis的链表实际为双向无环链表。
3. `listNode`中的数据域为一个`void`指针，按照指针特性是可以随便存东西的，个人感觉算是一种C语言泛型编程的技巧。
4. `list`中主要储存了表头节点指针、表尾节点指针以及一个长度计数器。表头表尾指针直接指向实际表头表尾，没有像一些实现指向一个内部头尾节点。长度计数器没什么好说的，是一个`unsigned long`类型。
5. `list`中还含有三个函数指针，分别指向节点复制函数、节点释放函数、节点值对比函数。
6. `listIter`迭代器里只含有一个迭代方向(int)以及一个指针指向当前的迭代到的节点。

&emsp;
## 二、内存分配
&emsp;&emsp;就源码来看，链表这个数据结构的内存分配相关内容还是蛮常规的。<br>
&emsp;&emsp;链表的内存申请主要就是管理者`list`的申请以及链表节点`listNode`的申请，redis中这两者的内存申请均使用自己封装的`zmalloc`函数，可以看成一个简单封装的`malloc`。因为链表的内存不必要全部连续，所以内存上的讲究就没太多，总体都比较简单。

&emsp;
## 三、一些值得一提的源码
### 1. 基本定义
```cpp
typedef struct listNode {

    // 前置节点
    struct listNode *prev;

    // 后置节点
    struct listNode *next;

    // 节点的值
    void *value;

} listNode;//链表节点

// 从表头向表尾进行迭代
#define AL_START_HEAD 0
// 从表尾到表头进行迭代
#define AL_START_TAIL 1

typedef struct listIter {

    // 当前迭代到的节点
    listNode *next;

    // 迭代的方向
    int direction;

} listIter;//链表迭代器

typedef struct list {

    // 表头节点
    listNode *head;

    // 表尾节点
    listNode *tail;

    // 节点值复制函数
    void *(*dup)(void *ptr);

    // 节点值释放函数
    void (*free)(void *ptr);

    // 节点值对比函数
    int (*match)(void *ptr, void *key);

    // 链表所包含的节点数量
    unsigned long len;

} list;//链表管理者
```
&emsp;&emsp;朴实无华的定义，`void*`算是C语言的一个特色吧。

&emsp;
### 2. 一堆宏
```cpp
// 返回给定链表所包含的节点数量
#define listLength(l) ((l)->len)
// 返回给定链表的表头节点
#define listFirst(l) ((l)->head)
// 返回给定链表的表尾节点
#define listLast(l) ((l)->tail)
// 返回给定节点的前置节点
#define listPrevNode(n) ((n)->prev)
// 返回给定节点的后置节点
#define listNextNode(n) ((n)->next)
// 返回给定节点的值
#define listNodeValue(n) ((n)->value)

// 将链表 l 的值复制函数设置为 m
#define listSetDupMethod(l,m) ((l)->dup = (m))
// 将链表 l 的值释放函数设置为 m
#define listSetFreeMethod(l,m) ((l)->free = (m))
// 将链表的对比函数设置为 m
#define listSetMatchMethod(l,m) ((l)->match = (m))

// 返回给定链表的值复制函数
#define listGetDupMethod(l) ((l)->dup)
// 返回给定链表的值释放函数
#define listGetFree(l) ((l)->free)
// 返回给定链表的值对比函数
#define listGetMatchMethod(l) ((l)->match)
```
&emsp;&emsp;redis的链表中涉及到的get和set操作基本上都使用宏来定义了。这种不太健康的宏函数感觉也算是C语言的一种特点吧。

&emsp;
### 3. 其余的一些函数
```cpp
list *listAddNodeHead(list *list, void *value);
list *listAddNodeTail(list *list, void *value);
list *listInsertNode(list *list, listNode *old_node, void *value, int after);
void listDelNode(list *list, listNode *node);
```
&emsp;&emsp;链表节点支持指定头插尾插和指定插，以及指定地址删除，都是链表常规操作。
```cpp
list *listDup(list *orig);
listNode *listSearchKey(list *list, void *key);
listNode *listIndex(list *list, long index);
void listRotate(list *list);
```
&emsp;&emsp;除了基础操作外还提供了常规的复制、搜索、求索引、旋转操作。