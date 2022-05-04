# 简单动态字符串sds
## 一、小结
1. sds结构体内含有`len`与`free`字段，故可以以常数时间获取长度与空余长度；内容则是使用一个char指针`buf`来指向。个人感觉和std::string思路相似。缺点就是需要多两个int变量，存小字符串时空间占用更大。
2. `len`字段用于确保二进制安全。即sds并非以常规C语言字符串判尾标准`\0`来判定，而是使用`len`字段来确定字符串长度。主要是避免redis保存二进制内容时其中的`\0`造成影响。但是其实sds的末尾也是有一个`\0`，这主要是为了尽量兼容C字符串风格。另外这个`\0`是不包括在`len`里的，即一个sds的`buf`区大小实际为`len`+`free`+`\0`。
3. `free`字段可以有效防止溢出。即在进行copy等操作时先对`free`字段进行比较判定即可避免溢出的情况。
4. `buf`指针的设计使得sds可以兼容部分C库。即可以传入`sds->buf`来使用C库常见函数。

&emsp;
## 二、内存分配
&emsp;&emsp;sds的内存分配是其实现"动态"的关键，主要是通过空间预分配和惰性释放来实现。其实感觉和STL里很多容器的思路很相似，只是细节上有些许差异。

### 1. 空间预分配
&emsp;&emsp;和`std::vector`很相似，只不过vector的扩容是基于当前的占用空间，而sds的扩容是基于更改后的占用空间(即更改后的`len`字段大小)。<br>
&emsp;&emsp;一般来讲，vector的扩容因子一般为固定的2或者1.5；而sds却有不同。sds在扩容时依据修改后大小分为两种策略：
* 当修改后sds`len`长度小于1MB时，会分配等同于`len`字段的未使用空间，即修改后的状态为`free`=`len`。
* 当修改后sds`len`长度大于等于1MB时，会分配1MB的未使用空间，即修改后的状态为`free`=1MB。

&emsp;&emsp;简单来讲就是，当修改后长度较小时，会直接分配同等的未使用空间；而修改后长度较大时，则固定分配1MB的未使用空间。`buf`的实际长度为(`len`+`free`+1)byte。这样的扩容策略可以保证在连续增长N次字符串时，所需的内存重分配次数从必定N次降低为最多N次。

### 2. 惰性空间释放
&emsp;&emsp;简单来讲就是在缩短sds保存的字符串时，sds并不会直接释放掉不用的内存，而是使用`free`字段来记录，等待未来使用。当然sds也给了相应的API来确保可以释放空间，避免内存浪费。<br>
&emsp;&emsp;这种策略个人感觉还是蛮常见的，包括STL里的很多容器，在执行clear()方法后，其实也不会释放内存空间，即size清零，capacity不清零。redis追求速度，这种惰性释放我感觉也是在空间和速度上的一个取舍，现在计算机的内存往往是很大的，所以这样的策略也可也被接受。

&emsp;
## 三、一些值得一提的源码
### 1. sds定义
```cpp
struct sdshdr {
    // buf 中已占用空间的长度
    int len;

    // buf 中剩余可用空间的长度
    int free;

    // 数据空间
    char buf[];
};
```
&emsp;&emsp;朴实无华的sds定义，包含三个核心字段`len`、`free`、`buf`。

&emsp;
### 2. 新建sds
```cpp
sds sdsnewlen(const void *init, size_t initlen);
```
&emsp;&emsp;此函数用于新建sds，`init`为初始化内容，`initlen`为初始化长度。
```cpp
if (init) {
    // zmalloc 不初始化所分配的内存
    sh = zmalloc(sizeof(struct sdshdr)+initlen+1);
} else {
    // zcalloc 将分配的内存全部初始化为 0
    sh = zcalloc(sizeof(struct sdshdr)+initlen+1);
}
```
&emsp;&emsp;此函数内会根据有没有初始化内容来调用不同的alloc函数分配内存。分配的内存大小为`sizeof(struct sdshdr)+initlen+1`，即一个sds结构体大小+`initlen`+1byte(`\0`)。<br>
&emsp;&emsp;`zmalloc`和`zcalloc`是redis自己实现的alloc函数。简单看了一眼`zmalloc`是对glic里`malloc`的封装，`zcalloc`则是对`calloc`的封装，两者都涉及到字节对齐和空间记录相关的优化。目前暂时不准备进一步了解，所以就此打住，待以后再进行记录探索。
```cpp
// 设置初始化长度
sh->len = initlen;
// 新 sds 不预留任何空间
sh->free = 0;
// T = O(N)
if (initlen && init)
    memcpy(sh->buf, init, initlen);
// 以 \0 结尾
sh->buf[initlen] = '\0';

// 返回 buf 部分，而不是整个 sdshdr
return (char*)sh->buf;
```
&emsp;&emsp;可以看到，一个新的sds是没有预留空间的，即`free`=0。这主要是之前alloc内存的时候，只分配了`initlen`长度的内存，并且全部属于`len`范围。此外内存初始化是拿memcpy做的(如果有初始化内容)，所以时间复杂度是O(N)。还有一些小细节，通过注释也可看到，huangz大佬的注释确实蛮细的。

&emsp;
### 3. 惰性删除
```cpp
void sdsclear(sds s) {

    // 取出 sdshdr
    struct sdshdr *sh = (void*) (s-(sizeof(struct sdshdr)));

    // 重新计算属性
    sh->free += sh->len;
    sh->len = 0;

    // 将结束符放到最前面（相当于惰性地删除 buf 中的内容）
    sh->buf[0] = '\0';
}
```
&emsp;&emsp;可以看到，空间并未被释放，而是直接进行三连 —— `free`扩展 + `len`置零 + `\0`前移。

&emsp;
### 4. 空间分配
```cpp
#define SDS_MAX_PREALLOC (1024*1024)//1MB

sds sdsMakeRoomFor(sds s, size_t addlen) {

    struct sdshdr *sh, *newsh;

    // 获取 s 目前的空余空间长度
    size_t free = sdsavail(s);

    size_t len, newlen;

    // s 目前的空余空间已经足够，无须再进行扩展，直接返回
    if (free >= addlen) return s;

    // 获取 s 目前已占用空间的长度
    len = sdslen(s);
    sh = (void*) (s-(sizeof(struct sdshdr)));

    // s 最少需要的长度
    newlen = (len+addlen);

    // 根据新长度，为 s 分配新空间所需的大小
    if (newlen < SDS_MAX_PREALLOC)
        // 如果新长度小于 SDS_MAX_PREALLOC 
        // 那么为它分配两倍于所需长度的空间
        newlen *= 2;
    else
        // 否则，分配长度为目前长度加上 SDS_MAX_PREALLOC
        newlen += SDS_MAX_PREALLOC;
    // T = O(N)
    newsh = zrealloc(sh, sizeof(struct sdshdr)+newlen+1);

    // 内存不足，分配失败，返回
    if (newsh == NULL) return NULL;

    // 更新 sds 的空余长度
    newsh->free = newlen - len;

    // 返回 sds
    return newsh->buf;
}
```
&emsp;&emsp;注释还是蛮详细的。`sdsavail()`函数其实就是一个指针向前偏移获取sds结构体地址。然后如果扩容时如果空间不够的话，就直接`realloc`一块新的内存来存内容，来完成扩容操作。<br>
&emsp;&emsp;所以这个申请的其实也是连续内存，本来还想着后续分配内存时只会重分配`buf`部分，但从这里的源码来看，会直接重申请全部的sds，来保证内存连续。所以之前猜测的重分配可能会导致外碎片问题基本也就不会出现了。
