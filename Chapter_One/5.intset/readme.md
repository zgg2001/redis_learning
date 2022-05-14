# 整数集合
## 一、总结
1. 整数集合是集合键的底层实现之一，当一个集合只包含整数值元素并且集合元素不多时，redis就会实验整数集合作为集合键的底层实现。否则redis就会使用`dict`作为底层数据结构。
2. 整数集合的底层实现为数组，并以有序、无重复的方式来保存集合元素。而当此数组类型不足以储存新元素时，整数集合会改变数组类型，并进行**扩容、重排、添加**三连操作，这个过程被称为**升级**。
3. 升级操作为整数集合带来了操作上的灵活性，并且尽可能的节省了内存。
4. 整数集合只支持升级操作，不支持降级操作。

&emsp;
## 二、整数集合的数据结构
```cpp
typedef struct intset {
    
    // 编码方式
    uint32_t encoding;

    // 集合包含的元素数量
    uint32_t length;

    // 保存元素的数组
    int8_t contents[];

} intset;
```
&emsp;&emsp;如上所示，整数集合底层实现为一个`int8_t`类型的数组，此外有两个`uint32_t`类型的元素分别统计编码和数量。<br>
&emsp;&emsp;但是需要注意的是，这里`int8_t`数组并不实际储存`int8_t`类型的值，只是开了`int8_t`类型的数组。实际取值是根据编码方式和数组下标来进行取值，下面会列出相关取值源码。
```cpp
/* Return the value at pos, given an encoding. 
 *
 * 根据给定的编码方式 enc ，返回集合的底层数组在 pos 索引上的元素。
 *
 * T = O(1)
 */
static int64_t _intsetGetEncoded(intset *is, int pos, uint8_t enc) {
    int64_t v64;
    int32_t v32;
    int16_t v16;

    // ((ENCODING*)is->contents) 首先将数组转换回被编码的类型
    // 然后 ((ENCODING*)is->contents)+pos 计算出元素在数组中的正确位置
    // 之后 member(&vEnc, ..., sizeof(vEnc)) 再从数组中拷贝出正确数量的字节
    // 如果有需要的话， memrevEncifbe(&vEnc) 会对拷贝出的字节进行大小端转换
    // 最后将值返回
    if (enc == INTSET_ENC_INT64) {
        memcpy(&v64,((int64_t*)is->contents)+pos,sizeof(v64));
        memrev64ifbe(&v64);
        return v64;
    } else if (enc == INTSET_ENC_INT32) {
        memcpy(&v32,((int32_t*)is->contents)+pos,sizeof(v32));
        memrev32ifbe(&v32);
        return v32;
    } else {
        memcpy(&v16,((int16_t*)is->contents)+pos,sizeof(v16));
        memrev16ifbe(&v16);
        return v16;
    }
}
```
&emsp;&emsp;如上，可以看到实际取值是通过`memcpy`来copy出相关内存，而这块内存的偏移量则会根据编码方式而变化。所以`int8_t`数组实际上并不储存`int8_t`元素，只是为了标记内存。<br>
&emsp;&emsp;另外可以看到在`memcpy`后进行了`memrevxxifbe`操作，这个操作是为了进行大小端转换，详情可以看一下这位大佬写的详解：[点我跳转](https://www.twblogs.net/a/5d06eb91bd9eee1ede038edb)

&emsp;
## 三、整数集合的升级操作
```cpp
/* Upgrades the intset to a larger encoding and inserts the given integer. 
 *
 * 根据值 value 所使用的编码方式，对整数集合的编码进行升级，
 * 并将值 value 添加到升级后的整数集合中。
 *
 * 返回值：添加新元素之后的整数集合
 *
 * T = O(N)
 */
static intset *intsetUpgradeAndAdd(intset *is, int64_t value) {
    
    // 当前的编码方式
    uint8_t curenc = intrev32ifbe(is->encoding);

    // 新值所需的编码方式
    uint8_t newenc = _intsetValueEncoding(value);

    // 当前集合的元素数量
    int length = intrev32ifbe(is->length);

    // 根据 value 的值，决定是将它添加到底层数组的最前端还是最后端
    // 注意，因为 value 的编码比集合原有的其他元素的编码都要大
    // 所以 value 要么大于集合中的所有元素，要么小于集合中的所有元素
    // 因此，value 只能添加到底层数组的最前端或最后端
    int prepend = value < 0 ? 1 : 0;

    /* First set new encoding and resize */
    // 更新集合的编码方式
    is->encoding = intrev32ifbe(newenc);
    // 根据新编码对集合（的底层数组）进行空间调整
    // T = O(N)
    is = intsetResize(is,intrev32ifbe(is->length)+1);

    /* Upgrade back-to-front so we don't overwrite values.
     * Note that the "prepend" variable is used to make sure we have an empty
     * space at either the beginning or the end of the intset. */
    // 根据集合原来的编码方式，从底层数组中取出集合元素
    // 然后再将元素以新编码的方式添加到集合中
    // 当完成了这个步骤之后，集合中所有原有的元素就完成了从旧编码到新编码的转换
    // 因为新分配的空间都放在数组的后端，所以程序先从后端向前端移动元素
    // 举个例子，假设原来有 curenc 编码的三个元素，它们在数组中排列如下：
    // | x | y | z | 
    // 当程序对数组进行重分配之后，数组就被扩容了（符号 ？ 表示未使用的内存）：
    // | x | y | z | ? |   ?   |   ?   |
    // 这时程序从数组后端开始，重新插入元素：
    // | x | y | z | ? |   z   |   ?   |
    // | x | y |   y   |   z   |   ?   |
    // |   x   |   y   |   z   |   ?   |
    // 最后，程序可以将新元素添加到最后 ？ 号标示的位置中：
    // |   x   |   y   |   z   |  new  |
    // 上面演示的是新元素比原来的所有元素都大的情况，也即是 prepend == 0
    // 当新元素比原来的所有元素都小时（prepend == 1），调整的过程如下：
    // | x | y | z | ? |   ?   |   ?   |
    // | x | y | z | ? |   ?   |   z   |
    // | x | y | z | ? |   y   |   z   |
    // | x | y |   x   |   y   |   z   |
    // 当添加新值时，原本的 | x | y | 的数据将被新值代替
    // |  new  |   x   |   y   |   z   |
    // T = O(N)
    while(length--)
        _intsetSet(is,length+prepend,_intsetGetEncoded(is,length,curenc));

    /* Set the value at the beginning or the end. */
    // 设置新值，根据 prepend 的值来决定是添加到数组头还是数组尾
    if (prepend)
        _intsetSet(is,0,value);
    else
        _intsetSet(is,intrev32ifbe(is->length),value);

    // 更新整数集合的元素数量
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);

    return is;
}
```
&emsp;&emsp;从上面的源码部分可以看出，数组集合的升级操作主要分成两步：第一步进行空间扩容，第二步进行数据的重设置。
```cpp
/* Resize the intset 
 *
 * 调整整数集合的内存空间大小
 *
 * 如果调整后的大小要比集合原来的大小要大，
 * 那么集合中原有元素的值不会被改变。
 *
 * 返回值：调整大小后的整数集合
 *
 * T = O(N)
 */
static intset *intsetResize(intset *is, uint32_t len) {

    // 计算数组的空间大小
    uint32_t size = len*intrev32ifbe(is->encoding);

    // 根据空间大小，重新分配空间
    // 注意这里使用的是 zrealloc ，
    // 所以如果新空间大小比原来的空间大小要大，
    // 那么数组原有的数据会被保留
    is = zrealloc(is,sizeof(intset)+size);

    return is;
}
```
&emsp;&emsp;这部分主要就是进行空间的重分配。这里使用redis自己的重封装的`zrealloc`来进行再分配，所以原有数据会被保留。
```cpp
/* Set the value at pos, using the configured encoding. 
 *
 * 根据集合的编码方式，将底层数组在 pos 位置上的值设为 value 。
 *
 * T = O(1)
 */
static void _intsetSet(intset *is, int pos, int64_t value) {

    // 取出集合的编码方式
    uint32_t encoding = intrev32ifbe(is->encoding);

    // 根据编码 ((Enc_t*)is->contents) 将数组转换回正确的类型
    // 然后 ((Enc_t*)is->contents)[pos] 定位到数组索引上
    // 接着 ((Enc_t*)is->contents)[pos] = value 将值赋给数组
    // 最后， ((Enc_t*)is->contents)+pos 定位到刚刚设置的新值上 
    // 如果有需要的话， memrevEncifbe 将对值进行大小端转换
    if (encoding == INTSET_ENC_INT64) {
        ((int64_t*)is->contents)[pos] = value;
        memrev64ifbe(((int64_t*)is->contents)+pos);
    } else if (encoding == INTSET_ENC_INT32) {
        ((int32_t*)is->contents)[pos] = value;
        memrev32ifbe(((int32_t*)is->contents)+pos);
    } else {
        ((int16_t*)is->contents)[pos] = value;
        memrev16ifbe(((int16_t*)is->contents)+pos);
    }
}
```
&emsp;&emsp;这部分是设置值相关的操作，就是根据编码格式计算偏移量然后进行赋值。