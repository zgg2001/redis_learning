# 一、前言
&emsp;&emsp;字典是目前使用率蛮高的一种KV存储数据结构，简单说就是一个**key**对应一个**value**，并且可以确保通过**key**可以快速的获取**value**。<br>
&emsp;&emsp;字典的具体实现还是有很大的差异的。在C++中一般字典结构被称为**map**，目前stl常用的字典结构有`std::map`和`std::unordered_map`，前者的底层实现是红黑树，后者的底层实现为哈希表，两者都可以实现高效率的查询/插入操作。<br>
&emsp;&emsp;回归到redis上，redis中字典的出场率很高，毕竟redis本来就是一个KV内存数据库。可以说其底层就是使用字典来实现的，而常规的增删查改也是基于字典来进行的。但是C语言中是没有内置字典数据结构的，所以redis自己构建了字典实现。

&emsp;
# 二、redis字典的实现思路
&emsp;&emsp;redis的字典底层实现采用哈希表，总体思路和C++里的`std::unordered_map`思路差不多，简单来说都是**哈希表+开链法**。但是由于C语言里也没有自己的`std::vector`，所以redis的字典存储空间也是需要自己通过`malloc/free`管理。<br>
&emsp;&emsp;从具体实现上来看，redis字典主要由三部分组成：字典部分、哈希表部分、哈希表节点部分。从关系上来看，字典部分含有两张哈希表，而哈希表中含有若干哈希表节点。<br>
&emsp;&emsp;redis的字典思路其实还算是简单，但是其中很多策略和细节的实现我都蛮感兴趣的，所以还是会细看一下。其中个人比较感兴趣的部分有**开链法的代码部分**以及**渐进式rehash的实现部分**。

&emsp;
# 三、实现源码分析
## 1. 哈希表节点数据结构
```c
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
&emsp;&emsp;如上为哈希表节点的数据结构，可以说是字典结构里的最底层数据结构。简而言之一个哈希表节点由常规的`key`、`value`以及一个`next`指针组成，还是比较清晰的。<br>
&emsp;&emsp;`key`的类型为`void`指针，来实现存各式各样的内容。redis字典的键部分一般都是字符串对象，所以就没有像值部分那样使用联合体来特化内容。<br>
&emsp;&emsp;`value`部分使用一个联合体，其中有`void*`类型和有/无符号64位`int`类型，另外新版本的redis中好像也添加了高精度浮点数`double`类型。这里使用联合体主要还是为了节省内存，避免存数值内容时的内存浪费。<br>
&emsp;&emsp;`next`指针部分算是开链法的具体实现，即通过这个指针来把哈希冲突的节点给连接起来，来解决哈希冲突问题。

## 2. 哈希表数据结构
```c
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
&emsp;&emsp;如上为哈希表的数据结构，`dictht`通过一个`dictEntry`指针数组来建表。除了基础的表结构，此数据结构中也存放了已使用大小、总大小和大小掩码。<br>
&emsp;&emsp;首先是二级指针`table`，用于指向一个哈希表节点数组，来保存哈希表节点。所以这部分内存实际上是由字典直接进行管理的，不像是STL里使用`std::vector`当底层。<br>
&emsp;&emsp;然后说一下`sizemask`这个属性，其值总是等于`size - 1`，其实也就是指针数组的下标范围(0 ~ size - 1)，相关公式为`index = hash & sizemask`，来**避免索引值超出数组范围**。redis字典的扩容策略要求哈希表大小全部为2的n次方，所以这里进行减一操作就可以获取范围掩码了，还是挺巧妙的。<br>
&emsp;&emsp;剩下的两个属性`size`和`used`就比较常规了，没什么好说的。

## 3. 字典数据结构
```c
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
&emsp;&emsp;如上为字典的数据结构。首先看最重要的，属性`ht`用于存放哈希表，而且可以看到是用大小为2的数组存的，主要就是为了方便进行**渐进式rehash操作**，而下面的`rehashidx`属性用来配合记录rehash进度，rehash部分下面会进行具体分析。<br>
&emsp;&emsp;其次`privdata`属性是一个`void*`指针，指向一些特定参数，这些参数用于配合`dictType`里的回调函数。<br>
&emsp;&emsp;接下来来看`type`属性，其类型为`dictType`指针。具体来看内容，`dictType`结构体里存了一组回调函数，其结构体内容如下：
```c
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
&emsp;&emsp;可以看到是一组用于操作指定类型键值对的函数，不同的字典配备函数也可能不同，算是为了方便进行泛型编程吧。

## 4. 渐进式rehash部分
&emsp;&emsp;首先讲一下redis字典的rehash策略，redis中为了防止在进行扩展/收缩的rehash时，由于数据过多造成服务器停止服务，采用了渐进式rehash思路。即rehash不一次性进行完毕，而是分多次、渐进式的对键值对进行rehash操作。<br>
&emsp;&emsp;首先实现渐进式rehash的基础就是得有可记录的、独立的两张新表，对此redis数据结构`dict`以`dictht`数组的形式存放哈希表，数组大小为2。日常使用时使用`[0]`表，在扩展/收缩时，为`[1]`分配新表，并渐进式将`[0]`表上的键值对rehash分配到`[1]`表上。当rehash完成时，`[1]`表拥有所有键值对而`[0]`表为空，此时释放`[0]`表，并将`[1]`表设置`[0]`表，再将`[1]`表置空，此时渐进式rehash流程完全完成。<br>
&emsp;&emsp;还有一个值得一提的重点就是渐进式rehash的具体过程：redis字典数据结构`dict`中有一个rehash索引`rehashidx`，每当对rehash过程中的字典进行增删查改操作时，程序除了进行指定操作外，还会将`[0]`表在`rehashidx`索引上的所有键值对rehash到`[1]`表上，随后索引`rehashidx`值自增一。当渐进式rehash过程结束后，索引`rehashidx`置为-1，意为未在rehash过程中。<br>
&emsp;&emsp;此外，还需要注意的一个细节是渐进式rehash过程中增删查改的操作。在这个过程中，字典是同时使用`[0]`和`[1]`两张表的，所以删、查、改操作会同时在两张表上进行，来保证不会漏数据。例如查找一个键时，会先在`[0]`表上找，如果没有则会去`[1]`表上找，找不到再返回。而增操作则会直接将新键值对放在`[1]`表上，`[0]`表不进行任何添加操作，这主要是为了保证`[0]`表数据只减不增，并随着rehash操作最终变成空表。<br>
&emsp;&emsp;然后来看一下具体的实现源码：
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
&emsp;&emsp;渐进式rehash的每一步都会把一个索引处内的一条链表rehash至`[1]`表上，其中涉及到索引更新、计数器更新和置空等操作。<br>

**最后看一下各种情况下进行rehash的源码部分：**
```c
static void _dictRehashStep(dict *d) {
    if (d->iterators == 0) dictRehash(d,1);
}
```
&emsp;&emsp;首先是封装好的进行单次rehash操作的函数。
```c
dictEntry *dictAddRaw(dict *d, void *key)
{
    ...
    // 如果条件允许的话，进行单步 rehash
    // T = O(1)
    if (dictIsRehashing(d)) _dictRehashStep(d);
    ...
```
&emsp;&emsp;尝试插入键会触发单步rehash。
```c
static int dictGenericDelete(dict *d, const void *key, int nofree)
{
   	...
    // 进行单步 rehash ，T = O(1)
    if (dictIsRehashing(d)) _dictRehashStep(d);
    ...
```
&emsp;&emsp;查找删除节点会触发单步rehash。
```c
dictEntry *dictFind(dict *d, const void *key)
{
	...
    // 如果条件允许的话，进行单步 rehash
    if (dictIsRehashing(d)) _dictRehashStep(d);
    ...	
```
&emsp;&emsp;只进行节点查找也会触发单步rehash。
```c
dictEntry *dictGetRandomKey(dict *d)
{
	...
    // 进行单步 rehash
    if (dictIsRehashing(d)) _dictRehashStep(d);
    ...
```
&emsp;&emsp;随机返回任一节点也会触发单步rehash。

## 5. 扩容部分
&emsp;&emsp;首先来讲一下redis字典的空间分配策略，总结哈希表空间变化规律如下：
* 当执行扩展操作时：新的大小为第一个大于等于**当前已使用大小二倍**的2的n次方。例如当前已使用大小为4，执行扩容操作，扩容后哈希表总大小为8(8 >= 4 * 2)。
* 当执行收缩操作时：新的大小为第一个大于等于**当前已使用大小**的2的n次方。例如当前已使用大小为3，执行收缩操作，收缩后哈希表总大小为4(4 >= 3)。

&emsp;&emsp;哈希表扩展和收缩的情况如下：(负载因子：已使用/总量)
* 当负载因子**小于**0.1时，进行收缩操作。
* 当未执行BGSAVE/BGREWRITEAOF命令时，负载因子**大于等于**1时，进行扩展操作。
* 当在执行BGSAVE/BGREWRITEAOF命令时，负载因子**大于等于**5时，进行扩展操作。

&emsp;&emsp;当执行BGSAVE/BGREWRITEAOF时提高扩展所需负载因子，主要是因为执行这两个命令时都需要redis创建子进程，而此时进行rehash操作可能会触发子进程的"写时复制"机制。所以此时减少rehash操作即可避免不必要的内存写入操作，最大限度的节约内存。<br>
&emsp;&emsp;然后来看一下具体的实现源码：
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

## 6. 开链法解决哈希冲突部分
&emsp;&emsp;最后我决定看一下开链法的具体实现，毕竟这么有名的思路，看一下redis源码里是怎么写的。首先看一下新增节点的源码部分：
```c
dictEntry *dictAddRaw(dict *d, void *key)
{
    int index;
    dictEntry *entry;
    dictht *ht;

    // 如果条件允许的话，进行单步 rehash
    // T = O(1)
    if (dictIsRehashing(d)) _dictRehashStep(d);

    /* Get the index of the new element, or -1 if
     * the element already exists. */
    // 计算键在哈希表中的索引值
    // 如果值为 -1 ，那么表示键已经存在
    // T = O(N)
    if ((index = _dictKeyIndex(d, key)) == -1)
        return NULL;

    // T = O(1)
    /* Allocate the memory and store the new entry */
    // 如果字典正在 rehash ，那么将新键添加到 1 号哈希表
    // 否则，将新键添加到 0 号哈希表
    ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];
    // 为新节点分配空间
    entry = zmalloc(sizeof(*entry));
    // 将新节点插入到链表表头
    entry->next = ht->table[index];
    ht->table[index] = entry;
    // 更新哈希表已使用节点数量
    ht->used++;

    /* Set the hash entry fields. */
    // 设置新节点的键
    // T = O(1)
    dictSetKey(d, entry, key);

    return entry;
}
```
&emsp;&emsp;太长不看，截取出关键部分：
```c
//hash取下标
index = _dictKeyIndex(d, key);
// 将新节点插入到链表表头
entry = zmalloc(sizeof(*entry));//申请新结点内存
entry->next = ht->table[index];//执行头插操作
ht->table[index] = entry;
```
&emsp;&emsp;所以实际上redis的开链法实现就是配合着数据结构里的`next`指针，简单的把每个新节点**头插**在对应下标的链表上，非常的简单，也就几行代码，完全没有进行任何哈希冲突方面的判定。

&emsp;
# 四、总结
&emsp;&emsp;redis的字典设计的还是非常巧妙的，尤其是其中的渐进式rehash实现。通过这次看具体代码我也是学到了很多新思路，尤其是看源码前完全没想到开链法实现竟然这么简单。<br>
&emsp;&emsp;就个人的感觉来看，redis的代码写的还是蛮清晰简便的，不像C++标准库套来套去非常的恶心。我感觉可能主要是因为C语言比较灵活，没有搞C++强类型那一套，所以通过`void*`就可以非常简便的实现泛型编程。其次可能就是redis的代码风格比较精简，确实读起来蛮舒服的（**当然也是huangz大佬的书写的好**）。