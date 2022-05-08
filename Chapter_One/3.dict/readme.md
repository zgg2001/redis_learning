# 字典
## 一、总结
1. redis的很大一部分内容都是建立在字典上进行的，redis中的字典数据结构层级关系为：字典`dict` -> 哈希表`dictht` -> 哈希表节点`dictEntry`，三者由上到下依次关联。此外redis的hash算法使用的是MurmurHash2。
2. redis中哈希表的hash冲突解决方法为开链法，依靠哈希表节点`dictEntry`实现头插单向链表，但是并没有像STL哈希表一样在链表过长时进行红黑树优化。
3. redis哈希表的扩容策略大致为依据当前键值对数量，扩展/收缩哈希表大小为一个合适的值，这个值必定为2的n次方。
4. redis中字典采用渐进式rehash方法，来避免服务器因为rehash而无法处理业务操作。
5. 另外从字典这部分的源码里(dict.c)，可以看到有一部分函数是私有函数(1389行后)，C语言中的私有函数使用关键字`static`达成。

&emsp;
## 二、内存分配
&emsp;&emsp;字典的内存分配相关的内容即为哈希表的扩展和收缩，所以这部分主要是记录一下redis中哈希表的扩展/收缩策略。字典的内存申请使用redis自己封装的`zmalloc`和`zcalloc`函数，可以大致看成带有额外内容的`malloc`和`calloc`函数。<br>
&emsp;&emsp;哈希表空间变化规律如下：
* 当执行扩展操作时：新的大小为第一个大于等于**当前已使用大小二倍**的2的n次方。例如当前已使用大小为4，执行扩容操作，扩容后哈希表总大小为8(8 >= 4 * 2)。
* 当执行收缩操作时：新的大小为第一个大于等于**当前已使用大小**的2的n次方。例如当前已使用大小为3，执行收缩操作，收缩后哈希表总大小为4(4 >= 3)。

&emsp;&emsp;哈希表扩展和收缩的情况如下：(负载因子：已使用/总量)
* 当负载因子**小于**0.1时，进行收缩操作。
* 当未执行BGSAVE/BGREWRITEAOF命令时，负载因子**大于等于**1时，进行扩展操作。
* 当在执行BGSAVE/BGREWRITEAOF命令时，负载因子**大于等于**5时，进行扩展操作。

&emsp;&emsp;当执行BGSAVE/BGREWRITEAOF时提高扩展所需负载因子，主要是因为执行这两个命令时都需要redis创建子进程，而此时进行rehash操作可能会触发子进程的"写时复制"机制。所以此时减少rehash操作即可避免不必要的内存写入操作，最大限度的节约内存。

&emsp;
## 三、渐进式rehash
&emsp;&emsp;redis中为了防止在进行扩展/收缩的rehash时，由于数据过多造成服务器停止服务，采用了渐进式rehash思路。即rehash不一次性进行完毕，而是分多次、渐进式的对键值对进行rehash操作。<br>
&emsp;&emsp;首先实现渐进式rehash的基础就是得有可记录的、独立的两张新表，对此redis数据结构`dict`以`dictht`数组的形式存放哈希表，数组大小为2。日常使用时使用`[0]`表，在扩展/收缩时，为`[1]`分配新表，并渐进式将`[0]`表上的键值对rehash分配到`[1]`表上。当rehash完成时，`[1]`表拥有所有键值对而`[0]`表为空，此时释放`[0]`表，并将`[1]`表设置`[0]`表，再将`[1]`表置空，此时渐进式rehash流程完全完成。<br>
&emsp;&emsp;还有一个值得一提的重点就是渐进式rehash的具体过程：redis字典数据结构`dict`中有一个rehash索引`rehashidx`，每当对rehash过程中的字典进行增删查改操作时，程序除了进行指定操作外，还会将`[0]`表在`rehashidx`索引上的所有键值对rehash到`[1]`表上，随后索引`rehashidx`值自增一。当渐进式rehash过程结束后，索引`rehashidx`置为-1，意为未在rehash过程中。<br>
&emsp;&emsp;此外，还需要注意的一个细节是渐进式rehash过程中增删查改的操作。在这个过程中，字典是同时使用`[0]`和`[1]`两张表的，所以删、查、改操作会同时在两张表上进行，来保证不会漏数据。例如查找一个键时，会先在`[0]`表上找，如果没有则会去`[1]`表上找，找不到再返回。而增操作则会直接将新键值对放在`[1]`表上，`[0]`表不进行任何添加操作，这主要是为了保证`[0]`表数据只减不增，并随着rehash操作最终变成空表。

&emsp;
## 四、一些值得一提的源码
### 1. 哈希表节点
```cpp
typedef struct dictEntry {
    
    // 键
    void *key;

    // 值
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
    } v;

    // 指向下个哈希表节点，形成链表
    struct dictEntry *next;

} dictEntry;
```
&emsp;&emsp;这是哈希表节点的数据结构，键是`void*`类型，值支持`void*`或是有符号/无符号64位`int`类型，算是C语言风格的泛型编程。<br>
&emsp;&emsp;此外还有个同类型指针来构建链表，即使用开链法解决hash冲突。

&emsp;
### 2. 哈希表
```cpp
typedef struct dictht {
    
    // 哈希表数组
    dictEntry **table;

    // 哈希表大小
    unsigned long size;
    
    // 哈希表大小掩码，用于计算索引值
    // 总是等于 size - 1
    unsigned long sizemask;

    // 该哈希表已有节点的数量
    unsigned long used;

} dictht;
```
&emsp;&emsp;这是哈希表节点的上层数据结构哈希表，`dictht`通过一个`dictEntry`指针数组来建表。除了基础的表结构，此数据结构中也存放了已使用大小、总大小和大小掩码。<br>
&emsp;&emsp;这个大小掩码，总是等于`size - 1`，其实也就是指针数组的下标范围(0 ~ size - 1)，相关公式为`index = hash & sizemask`，来避免超出数组范围。前面扩容策略要求大小全部为2的n次方，所以这里进行减一操作就可以获取范围源码了，还是挺巧妙的。

&emsp;
### 3. 字典
```cpp
typedef struct dict {

    // 类型特定函数
    dictType *type;

    // 私有数据
    void *privdata;

    // 哈希表
    dictht ht[2];

    // rehash 索引
    // 当 rehash 不在进行时，值为 -1
    int rehashidx; /* rehashing not in progress if rehashidx == -1 */

    // 目前正在运行的安全迭代器的数量
    int iterators; /* number of iterators currently running */

} dict;
```
&emsp;&emsp;这是字典的数据结构，可以看到哈希表是用数组存的，大小为2，就是为了方便前文提到的渐进式rehash操作，下面的`rehashidx`用来配合记录rehash进度。<br>
&emsp;&emsp;`dictType`结构体里里存了一组回调函数，内容如下：
```cpp
typedef struct dictType {

    // 计算哈希值的函数
    unsigned int (*hashFunction)(const void *key);

    // 复制键的函数
    void *(*keyDup)(void *privdata, const void *key);

    // 复制值的函数
    void *(*valDup)(void *privdata, const void *obj);

    // 对比键的函数
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);

    // 销毁键的函数
    void (*keyDestructor)(void *privdata, void *key);
    
    // 销毁值的函数
    void (*valDestructor)(void *privdata, void *obj);

} dictType;
```
&emsp;&emsp;可以看到是一些用于操作指定类型键值对的函数，不同的字典配备函数也可能不同，算是一种泛型编程吧。<br>
&emsp;&emsp;`privdata`类型是一个`void*`指针，指向一些特定参数，这些参数用于配合`dictType`里的回调函数。

&emsp;
### 4. 渐进式rehash部分
```cpp
int dictRehash(dict *d, int n) {

    // 只可以在 rehash 进行中时执行
    if (!dictIsRehashing(d)) return 0;

    // 进行 N 步迁移
    // T = O(N)
    while(n--) {
        dictEntry *de, *nextde;

        /* Check if we already rehashed the whole table... */
        // 如果 0 号哈希表为空，那么表示 rehash 执行完毕
        // T = O(1)
        if (d->ht[0].used == 0) {
            // 释放 0 号哈希表
            zfree(d->ht[0].table);
            // 将原来的 1 号哈希表设置为新的 0 号哈希表
            d->ht[0] = d->ht[1];
            // 重置旧的 1 号哈希表
            _dictReset(&d->ht[1]);
            // 关闭 rehash 标识
            d->rehashidx = -1;
            // 返回 0 ，向调用者表示 rehash 已经完成
            return 0;
        }

        /* Note that rehashidx can't overflow as we are sure there are more
         * elements because ht[0].used != 0 */
        // 确保 rehashidx 没有越界
        assert(d->ht[0].size > (unsigned)d->rehashidx);

        // 略过数组中为空的索引，找到下一个非空索引
        while(d->ht[0].table[d->rehashidx] == NULL) d->rehashidx++;

        // 指向该索引的链表表头节点
        de = d->ht[0].table[d->rehashidx];
        /* Move all the keys in this bucket from the old to the new hash HT */
        // 将链表中的所有节点迁移到新哈希表
        // T = O(1)
        while(de) {
            unsigned int h;

            // 保存下个节点的指针
            nextde = de->next;

            /* Get the index in the new hash table */
            // 计算新哈希表的哈希值，以及节点插入的索引位置
            h = dictHashKey(d, de->key) & d->ht[1].sizemask;

            // 插入节点到新哈希表
            de->next = d->ht[1].table[h];
            d->ht[1].table[h] = de;

            // 更新计数器
            d->ht[0].used--;
            d->ht[1].used++;

            // 继续处理下个节点
            de = nextde;
        }
        // 将刚迁移完的哈希表索引的指针设为空
        d->ht[0].table[d->rehashidx] = NULL;
        // 更新 rehash 索引
        d->rehashidx++;
    }

    return 1;
}
```
&emsp;&emsp;可以看到在渐进式rehash的过程里，碰到空索引会跳过并且会自动找下一个非空索引，直至完成指定步数或者`[0]`表为空。当`[0]`表为空时，进行释放、交换、重置三连，并标记渐进式rehash结束。<br>
&emsp;&emsp;渐进式rehash的每一步都会把一个索引处内的一条链表rehash至`[1]`表上，其中涉及到索引更新、计数器更新和置空等操作。

&emsp;
### 5. 扩容部分
```cpp
int dictExpand(dict *d, unsigned long size):

    // 新哈希表
    dictht n;
    
    // 根据 size 参数，计算哈希表的大小
    unsigned long realsize = _dictNextPower(size);

    /* the size is invalid if it is smaller than the number of
     * elements already inside the hash table */
    // 不能在字典正在 rehash 时进行
    // size 的值也不能小于 0 号哈希表的当前已使用节点
    if (dictIsRehashing(d) || d->ht[0].used > size)
        return DICT_ERR;

    /* Allocate the new hash table and initialize all pointers to NULL */
    // 为哈希表分配空间，并将所有指针指向 NULL
    n.size = realsize;
    n.sizemask = realsize-1;
    // T = O(N)
    n.table = zcalloc(realsize*sizeof(dictEntry*));
    n.used = 0;
```
&emsp;&emsp;可以看到这部分在获取下一个大小后进行了内存申请，申请了`realsize`个`dictEntry`指针大小的内存。
```cpp
    // 如果 0 号哈希表为空，那么这是一次初始化：
    // 程序将新哈希表赋给 0 号哈希表的指针，然后字典就可以开始处理键值对了。
    if (d->ht[0].table == NULL) {
        d->ht[0] = n;
        return DICT_OK;
    }

    // 如果 0 号哈希表非空，那么这是一次 rehash ：
    // 程序将新哈希表设置为 1 号哈希表，
    // 并将字典的 rehash 标识打开，让程序可以开始对字典进行 rehash
    d->ht[1] = n;
    d->rehashidx = 0;
    return DICT_OK;
```
&emsp;&emsp;这部分主要是为了进入rehash状态，当然如果是初始化就不用进rehash状态。