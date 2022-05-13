# 跳跃表
## 一、总结
1. 跳跃表是redis中经常被提起的一个数据结构，简单来说就是在普通链表上添加了多层索引来实现快速查找。跳跃表目前在redis中的唯一作用，就是作为有序集合类型的底层数据结构之一。
2. 跳跃表的优化主要体现于支持平均O(logN)、最坏O(N)复杂度的节点查找。除此外还可以通过顺序性操作来批量处理节点。
3. redis中跳跃表的实现主要是由跳表节点`zskiplistNode`和跳跃表`zskiplist`两个结构组成。其中`zskiplistNode`中定义了跳跃表节点的内容，而`zskiplist`中定义的内容则是为了方便管理表示跳跃表。总体而言跳跃表的这种**节点+管理**的结构和常规链表类型的结构也差不多。
4. 跳跃表中会根据节点的`score`来进行排序，也就是说跳跃表的数据是有序的。另外节点中的`score`相同时，会根据所储存的`sds`对象的字典序进行排序(从小到大)。

&emsp;
## 二、跳跃表定义与思路
### 1. 跳跃表节点
```cpp
typedef struct zskiplistNode {

    // 成员对象
    robj *obj;

    // 分值
    double score;

    // 后退指针
    struct zskiplistNode *backward;

    // 层
    struct zskiplistLevel {

        // 前进指针
        struct zskiplistNode *forward;

        // 跨度
        unsigned int span;

    } level[];

} zskiplistNode;
```
&emsp;&emsp;可以看到redis的跳跃表节点定义中，除了常规链表中的数据域外，存在一个后退指针和**一组**前进指针。跳跃表和普通双向链表的主要区别就是在这一组前进指针上。<br>
&emsp;&emsp;常规双向链表只有一个前进指针，指向下一个节点。而跳跃表则拥有一组前进指针，指向的不一定为下一个节点，则在查找节点时可能会出现"跳跃"情况，所以这种数据结构被称为跳跃表，个人感觉还是蛮形象的。所以跳跃表的时间优化就体现在查找时对一些节点的"跳跃"，来避免一个一个遍历，配合一定的"高度"生成策略，实现平均O(logN)的时间复杂度。<br>
&emsp;&emsp;在前进指针数据结构`zskiplistLevel`中，每个元素存在一个前进指针和一个跨度值，跨度值主要是为了计算节点排位的，即节点在链表中的位置。一个前进指针移动一次经过N个节点，则跨度值则为N。后退指针则是一个一个节点相连，主要是为了从表尾向表头遍历，遍历时间复杂度为O(N)。<br>
&emsp;&emsp;每当创建一个新节点时都会根据幂次定律来随机生成一个1到32的值来作为层结构数组`level`的大小，这个大小被称为高度。假设层结构数组中存在一个层数为N的的指针，则此指针指向下一个高度大于等于N的节点，不存在则指向`NULL`。由此即可实现跨节点相连的跳跃表。

&emsp;
### 2. 跳跃表
```cpp
typedef struct zskiplist {

    // 表头节点和表尾节点
    struct zskiplistNode *header, *tail;

    // 表中节点的数量
    unsigned long length;

    // 表中层数最大的节点的层数
    int level;

} zskiplist;
```
&emsp;&emsp;跳跃表的管理部分定义如上。总体来看和常规双向链表的管理思路差不多，头尾指针和一个长度统计，但是`zskiplist`多了一个最大层数的统计。<br>
&emsp;&emsp;此外相比常规普通双向链表，跳跃表是有头结点定义的，有一个层数为32的满`level[]`头结点来统领所有跳跃表节点。在无节点的情况下，`header`指向此头结点(`tail`指向`NULL`)。这样做的主要目的是为了方便**储存/寻找**各个高度的第一个节点，从而方便快速跳跃查找。

&emsp;
## 三、一些源码分析
### 1. 新建跳跃表节点/跳跃表
```cpp
zskiplistNode *zslCreateNode(int level, double score, robj *obj) {
    
    // 分配空间
    zskiplistNode *zn = zmalloc(sizeof(*zn)+level*sizeof(struct zskiplistLevel));

    // 设置属性
    zn->score = score;
    zn->obj = obj;

    return zn;
}
```
&emsp;&emsp;新建节点就是很简单的`zmalloc`一块内存，然后初始化设置相关属性即可。这里需要注意的是申请内存的时候，层数组部分的大小和传参`level`有关。<br>
&emsp;&emsp;但是这样有个问题就是通过节点无法判定此节点层数组的大小，但是可能也没这个需求，只需要知道整个跳跃表的最高层数即可。
```cpp
zskiplist *zslCreate(void) {
    int j;
    zskiplist *zsl;

    // 分配空间
    zsl = zmalloc(sizeof(*zsl));

    // 设置高度和起始层数
    zsl->level = 1;
    zsl->length = 0;

    // 初始化表头节点
    // T = O(1)
    zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL,0,NULL);
    for (j = 0; j < ZSKIPLIST_MAXLEVEL; j++) {
        zsl->header->level[j].forward = NULL;
        zsl->header->level[j].span = 0;
    }
    zsl->header->backward = NULL;

    // 设置表尾
    zsl->tail = NULL;

    return zsl;
}
```
&emsp;&emsp;新建跳跃表的过程如上，相比来说都比较简单，总体流程就是首先`zmalloc`申请内存并简单初始化，随后新建头结点并将32层前进指针和后退指针初始化，最后就是设置表尾。<br>
&emsp;&emsp;需要注意的是空跳跃表的表尾指针指向`NULL`，并不是指向头结点。

&emsp;
### 2. 新节点的层数生成
```cpp
/* Returns a random level for the new skiplist node we are going to create.
 *
 * 返回一个随机值，用作新跳跃表节点的层数。
 *
 * The return value of this function is between 1 and ZSKIPLIST_MAXLEVEL
 * (both inclusive), with a powerlaw-alike distribution where higher
 * levels are less likely to be returned. 
 *
 * 返回值介乎 1 和 ZSKIPLIST_MAXLEVEL 之间（包含 ZSKIPLIST_MAXLEVEL），
 * 根据随机算法所使用的幂次定律，越大的值生成的几率越小。
 *
 * T = O(N)
 */
#define ZSKIPLIST_P 0.25      /* Skiplist P = 1/4 */

int zslRandomLevel(void) {
    int level = 1;

    while ((random()&0xFFFF) < (ZSKIPLIST_P * 0xFFFF))
        level += 1;

    return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
}
```
&emsp;&emsp;新节点层数生成过程如上。简单来讲，就是初始层数为1，每进入下一层的概率为四分之一。按期望来看，生成4个节点，其中有一个2层节点和三个1层节点...<br>
&emsp;&emsp;按照这个幂次规律，可以保证高层数出现概率比底层数低，可以保证跳跃表整体层数的稳定。

&emsp;
### 3. 添加新节点
```cpp
/*
 * 创建一个成员为 obj ，分值为 score 的新节点，
 * 并将这个新节点插入到跳跃表 zsl 中。
 * 
 * 函数的返回值为新节点。
 *
 * T_wrost = O(N^2), T_avg = O(N log N)
 */
zskiplistNode *zslInsert(zskiplist *zsl, double score, robj *obj) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    unsigned int rank[ZSKIPLIST_MAXLEVEL];
    int i, level;

    redisAssert(!isnan(score));

    // 在各个层查找节点的插入位置
    // T_wrost = O(N^2), T_avg = O(N log N)
    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {

        /* store rank that is crossed to reach the insert position */
        // 如果 i 不是 zsl->level-1 层
        // 那么 i 层的起始 rank 值为 i+1 层的 rank 值
        // 各个层的 rank 值一层层累积
        // 最终 rank[0] 的值加一就是新节点的前置节点的排位
        // rank[0] 会在后面成为计算 span 值和 rank 值的基础
        rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];

        // 沿着前进指针遍历跳跃表
        // T_wrost = O(N^2), T_avg = O(N log N)
        while (x->level[i].forward &&
            (x->level[i].forward->score < score ||
                // 比对分值
                (x->level[i].forward->score == score &&
                // 比对成员， T = O(N)
                compareStringObjects(x->level[i].forward->obj,obj) < 0))) {

            // 记录沿途跨越了多少个节点
            rank[i] += x->level[i].span;

            // 移动至下一指针
            x = x->level[i].forward;
        }
        // 记录将要和新节点相连接的节点
        update[i] = x;
    }

    /* zslInsert() 的调用者会确保同分值且同成员的元素不会出现，
     * 所以这里不需要进一步进行检查，可以直接创建新元素。
     */

    // 获取一个随机值作为新节点的层数
    // T = O(N)
    level = zslRandomLevel();

    // 如果新节点的层数比表中其他节点的层数都要大
    // 那么初始化表头节点中未使用的层，并将它们记录到 update 数组中
    // 将来也指向新节点
    if (level > zsl->level) {

        // 初始化未使用层
        // T = O(1)
        for (i = zsl->level; i < level; i++) {
            rank[i] = 0;
            update[i] = zsl->header;
            update[i]->level[i].span = zsl->length;
        }

        // 更新表中节点最大层数
        zsl->level = level;
    }

    // 创建新节点
    x = zslCreateNode(level,score,obj);

    // 将前面记录的指针指向新节点，并做相应的设置
    // T = O(1)
    for (i = 0; i < level; i++) {
        
        // 设置新节点的 forward 指针
        x->level[i].forward = update[i]->level[i].forward;
        
        // 将沿途记录的各个节点的 forward 指针指向新节点
        update[i]->level[i].forward = x;

        /* update span covered by update[i] as x is inserted here */
        // 计算新节点跨越的节点数量
        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);

        // 更新新节点插入之后，沿途节点的 span 值
        // 其中的 +1 计算的是新节点
        update[i]->level[i].span = (rank[0] - rank[i]) + 1;
    }

    // 未接触的节点的 span 值也需要增一，这些节点直接从表头指向新节点
    // T = O(1)
    for (i = level; i < zsl->level; i++) {
        update[i]->level[i].span++;
    }

    // 设置新节点的后退指针
    x->backward = (update[0] == zsl->header) ? NULL : update[0];
    if (x->level[0].forward)
        x->level[0].forward->backward = x;
    else
        zsl->tail = x;

    // 跳跃表的节点计数增一
    zsl->length++;

    return x;
}
```
&emsp;&emsp;简单来说，在插入新结点前，首先要通过头结点来遍历所有已应用层，根据`score`找到每一层需要插入的位置，同时也统计了各层的排位`rank`。<br>
&emsp;&emsp;随后，随机取新节点层数并新建节点，这里需要注意当层数过高时则需进行边界处理。<br>
&emsp;&emsp;最后，根据第一步遍历的结果，将新节点`insert`至每一层对应的位置(设置前进指针和`span`部分)，并设置后退指针并更新节点计数。至此插入流程完成。

&emsp;
### 4. 根据排位查找
```cpp
/* Finds an element by its rank. The rank argument needs to be 1-based. 
 * 
 * 根据排位在跳跃表中查找元素。排位的起始值为 1 。
 *
 * 成功查找返回相应的跳跃表节点，没找到则返回 NULL 。
 *
 * T_wrost = O(N), T_avg = O(log N)
 */
zskiplistNode* zslGetElementByRank(zskiplist *zsl, unsigned long rank) {
    zskiplistNode *x;
    unsigned long traversed = 0;
    int i;

    // T_wrost = O(N), T_avg = O(log N)
    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {

        // 遍历跳跃表并累积越过的节点数量
        while (x->level[i].forward && (traversed + x->level[i].span) <= rank)
        {
            traversed += x->level[i].span;
            x = x->level[i].forward;
        }

        // 如果越过的节点数量已经等于 rank
        // 那么说明已经到达要找的节点
        if (traversed == rank) {
            return x;
        }

    }

    // 没找到目标节点
    return NULL;
}
```
&emsp;&emsp;这部分内容体现了跳跃表查找的效率，可以看到在查找时优先从高层开始"跳跃"遍历，随后逐渐降低层数回到依次遍历。