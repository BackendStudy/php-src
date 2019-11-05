# PHP7数组源码解析
> 数组是PHP中非常强大、灵活的一种数据类型.本次分享主要介绍一下数组的底层结构;如何对数组进行查找、写入、扩容操作;以及遇到Hash冲突时是怎样巧妙处理的.

## 1.zval结构体
在讲它的底层实现之前,我们先了解一下变量的结构zval.(为了减少篇幅,不重要的部分用...代替)


```
//本文源码均基于php7.1.26
struct _zval_struct {
    zend_value        value;      
    union {
        struct {
            ZEND_ENDIAN_LOHI_4(
                zend_uchar    type,     
                ...
        } v;
        uint32_t type_info;
    } u1;
    union {
        uint32_t     next;     
        ...
    } u2;
};

typedef union _zend_value {
    zend_long         lval;             
    double            dval;             
    zend_string      *str;
    zend_array       *arr;
    ...
} zend_value;
```

u1联合体中的type用来控制变量类型,类型之间的对应关系如下:

```
/* regular data types */
#define IS_UNDEF                    0
#define IS_NULL                     1
#define IS_FALSE                    2
#define IS_TRUE                     3
#define IS_LONG                     4
#define IS_DOUBLE                   5
#define IS_STRING                   6
#define IS_ARRAY                    7
#define IS_OBJECT                   8
#define IS_RESOURCE                 9
#define IS_REFERENCE                10
```

当type为7时变量类型为数组
u2联合体中的next用来解决hash冲突

## 2.HashTable的巧妙设计

PHP 数组的底层实现是散列表（也叫 hashTable )，散列表是根据键（Key）直接访问内存存储位置的数据结构，它的 key - value 之间存在一个映射函数，可以根据 key 通过映射函数得到的散列值直接索引到对应的 value 值，无需通过关键字比较，在理想情况下，不考虑散列冲突，散列表的查找效率是非常高的，时间复杂度是 O (1)。
从源码中我们可以看到 zend_array 的结构如下：


```
typedef struct _zend_array HashTable;

struct _zend_array {
    zend_refcounted_h gc;
    union {
        struct {
            ZEND_ENDIAN_LOHI_4(
                zend_uchar    flags,
                zend_uchar    nApplyCount,
                zend_uchar    nIteratorsCount,
                zend_uchar    consistency)
        } v;
        uint32_t flags;
    } u;
    uint32_t          nTableMask;
    Bucket           *arData;
    uint32_t          nNumUsed;
    uint32_t          nNumOfElements;
    uint32_t          nTableSize;
    uint32_t          nInternalPointer;
    zend_long         nNextFreeElement;
    dtor_func_t       pDestructor;
};
```

主要字段介绍:

* gc: 垃圾回收相关,记录了引用计数等信息.
* nTableSize: 数组占用的长度,2的n次方(最小为8).
* nTableMask :这个值在散列函数根据key的hash code映射元素的存储位置时用到,它的值实际就是nTableSize的负数,用位运算表示的话则为nTableMask=~nTableSize+1.
* nNumUsed,NNumOfElements:这两个成员的含义看起来非常相似,但是他们代表的含义是不同的,nNumUsed是指数组当前是用的Bucket数,但是这些Bucket并不都是数组的有效元素,比如当我们unset一个数组元素时并不会马上将其从数组中移除,而只是将这个元素的类型标为IS_UNDEF;nNumOfElements则是数组中有效元素的数量,所以nNumOfElements<=nNumUsed
* nNextFreeElement:当增加一个没有键值的元素时,如$arr[]='abc',nNextFreeElement就是该元素的键值,从0开始,每次这样赋值时就自增1
* 
HashTable结构中的 Bucket 即储存元素的数组，arData 指向数组的起始位置，使用映射函数对 key 值进行映射后可以得到偏移值，通过偏移值即可定位到元素存在哪个Bucket中。
Bucket 的数据结构如下：


```
typedef struct _Bucket {
    zval              val;
    zend_ulong        h;                /* hash value (or numeric index)   */
    zend_string      *key;              /* string key or NULL for numerics */
} Bucket;
```
* val 具体值,是个zval
* h  hash出来的值(算法为time33)
* *key 元素键值的地址

## 3.如何实现

php中实现散列表主要使用**存储元素数组**、**映射函数**和**散列表**
举个例子:
如果我们声明一个数组会发生什么


```
<?php
    $a['foo']=1;
    $a[]=2;
    $a['s']=3;
    $a['x']=4;
```

1. 初始化:_zend_hash_init()
2. 将每个元素对应的值依照顺序拷贝到 Bucket里。
3. 通过映射函数求值,根据算出来的值将其对应的bucket的地址写到该值对应的映射表里。
 
具体操作如下图所示:

![hash1](http://open.baolanbb.com/hash1.png)

目前这种映射是理想映射状态,因为在映射的过程中可能存在冲突,即多个元素nIndex相同,接下来会介绍映射函数,以及如何应对hash冲突

### 数组初始化:


```
ZEND_API void ZEND_FASTCALL _zend_hash_init(HashTable *ht, uint32_t nSize, dtor_func_t pDestructor, zend_bool persistent ZEND_FILE_LINE_DC)
{
    GC_REFCOUNT(ht) = 1;
    GC_TYPE_INFO(ht) = IS_ARRAY;//设置数据类型为数组
    ht->u.flags = (persistent ? HASH_FLAG_PERSISTENT : 0) | HASH_FLAG_APPLY_PROTECTION | HASH_FLAG_STATIC_KEYS;
    ht->nTableSize = zend_hash_check_size(nSize);//初始值为8
    ht->nTableMask = HT_MIN_MASK;//hash_array初始值为-2
    HT_SET_DATA_ADDR(ht, &uninitialized_bucket);
    ht->nNumUsed = 0;
    ht->nNumOfElements = 0;
    ht->nInternalPointer = HT_INVALID_IDX;
    ht->nNextFreeElement = 0;
    ht->pDestructor = pDestructor;
}
```

### 映射函数:

```
nIndex = h | ht->nTableMask;
```

将 key 经过 time33 算法生成的哈希值 h 和 nTableMask 进行或运算即可得出映射表的下标，其中 nTableMask 数值为 nTableSize 的负数。并且由于 nTableSize 的值为 2 的幂次方，所以 nTableMask 二进制位右侧全部为 0，保证了 h | ht->nTableMask 的取值范围会在 [-nTableSize, -1] 之间，正好在映射表的下标范围内.以nTableMask=-8为例看一下


```
111111111111111....1000 
101010101010111....1001
```
传统的方法是取余,也可以实现,按位或运算速度更快一些.

### 插入元素:


```
//_zend_hash_add_or_update_i
add_to_hash:
    idx = ht->nNumUsed++; //数组已使用 Bucket 数 +1
    ht->nNumOfElements++; // 数组有效元素数目 +1
    if (ht->nInternalPointer == HT_INVALID_IDX) {
        ht->nInternalPointer = idx;
    }
    zend_hash_iterators_update(ht, HT_INVALID_IDX, idx);
    p = ht->arData + idx;// p 为新元素对应的 Bucket的指针
    p->key = key;    //设置键名
    if (!ZSTR_IS_INTERNED(key)) {
        zend_string_addref(key);
        ht->u.flags &= ~HASH_FLAG_STATIC_KEYS;
        zend_string_hash_val(key);
    }
    p->h = h = ZSTR_H(key);//根据key计算h值
    ZVAL_COPY_VALUE(&p->val, pData);//将pData赋值给Bucket的val
    nIndex = h | ht->nTableMask;//计算映射下标
    Z_NEXT(p->val) = HT_HASH(ht, nIndex);//将原映射表中的内容赋值给新元素变量值的 u2.next成员,用于以后解决哈希冲突
    HT_HASH(ht, nIndex) = HT_IDX_TO_HASH(idx); //将映射表中的值设为 idx
    return &p->val;
```

### 哈希冲突:

散列表中通过哈希函数(nIndex = h | ht->nTableMask)计算出来的值可能相同,但是映射表一个位置只能存储一个元素,由此便发生了冲突.
其具体实现是：将冲突的 Bucket 串成链表，这样中间映射表映射出的就不是某一个元素，而是一个 Bucket 链表，通过散列函数定位到对应的 Bucket 链表时，需要遍历链表，逐个对比 Key 值，继而找到目标元素。而每个 Bucket 之间的链接则是将原 value 的下标保存到新 value 的 zval.u2.next 里，新 value 放在当前位置上，从而形成一个单向链表。

我们继续执行上边的例子:

![hash2](http://open.baolanbb.com/hash2.png)

当写入第三个元素时,通过键值's'算出nIndex也等于-7.于是映射表第七个位置的值变成了当前元素的下标2,
val.u2.next记录了映射表被替换的值:0

下面我们模拟一下查找$a['foo']的过程
1. 先通过hash函数计算出nIndex=-7,
2. 去映射表第7个格子中查找Bucket的下标,拿到里边的值2;
3. 然后去Bucket数组中找到下标为2的Bucket,对比一下key值,'s'!='foo',不是我们要找的
4. 此时继续找val.u2.next,值为0,
5. 去Bucket数组中找到下标为0的Bucket,对比key值,'foo'=='foo'.符合要求,查找结束
 

### 查找:
了解了哈希冲突的解决方式后,查找的过过程就比较简单了,首先通过哈希函数计算出key对应的散列值nIndex,然后根据散列值从中间映射表中得到存储元素在arData中的下标idx,接着根据idx从arData中取出Bucket,最后从取出的Bucket开始遍历,判断Bucket的key是否是要查找的key,如果是则终止遍历,否则继续根据zval.u2.next遍历比较


```
static zend_always_inline Bucket *zend_hash_find_bucket(const HashTable *ht, zend_string *key)
{
    zend_ulong h;
    uint32_t nIndex;
    uint32_t idx;
    Bucket *p, *arData;

    h = zend_string_hash_val(key);
    arData = ht->arData;
    nIndex = h | ht->nTableMask;
    //获取Bucket的存储位置     
    idx = HT_HASH_EX(arData, nIndex);
    //遍历
    while (EXPECTED(idx != HT_INVALID_IDX)) {  
        p = HT_HASH_TO_BUCKET_EX(arData, idx);
        if (EXPECTED(p->key == key)) { /* check for the same interned string */
            return p;
        } else if (EXPECTED(p->h == h) &&
             EXPECTED(p->key) &&
             //比较key长度
             EXPECTED(ZSTR_LEN(p->key) == ZSTR_LEN(key)) &&
             EXPECTED(memcmp(ZSTR_VAL(p->key), ZSTR_VAL(key), ZSTR_LEN(key)) == 0)) {
            return p;
        }
        //不匹配则继续遍历
        idx = Z_NEXT(p->val);//u2.next
    }
    return NULL;
}
```

### 扩容:

数组的容量在初始化时就已经确定了大小,就是nTableSize.当数组空间已满还要继续插入时,就会触发自动扩容机制，扩容后再执行插入。


```
#define ZEND_HASH_IF_FULL_DO_RESIZE(ht)             \
    if ((ht)->nNumUsed >= (ht)->nTableSize) {       \
        zend_hash_do_resize(ht);                    \
    }
```

并非每次BUcket数组满了都需要扩容,如果Bucket数组中IS_UNDEF元素的数量占较大比例时,就直接将IS_UNDEF元素删除并重新索引,以节省内存,下面我们看看zend_hash_do_resize函数:


```
static void ZEND_FASTCALL zend_hash_do_resize(HashTable *ht)
{

    IS_CONSISTENT(ht);
    HT_ASSERT(GC_REFCOUNT(ht) == 1);

    if (ht->nNumUsed > ht->nNumOfElements + (ht->nNumOfElements >> 5)) { /* additional term is there to amortize the cost of compaction */
           zend_hash_rehash(ht);//重新索引
    } else if (ht->nTableSize < HT_MAX_SIZE) {  //数组大小要小于最大限制0x04000000
        void *new_data, *old_data = HT_GET_DATA_ADDR(ht);
        uint32_t nSize = ht->nTableSize + ht->nTableSize;//扩大2倍,加法比乘法快
        Bucket *old_buckets = ht->arData;

        new_data = pemalloc(HT_SIZE_EX(nSize, -nSize), ht->u.flags & HASH_FLAG_PERSISTENT);//申请新数组内存
        ht->nTableSize = nSize;
        ht->nTableMask = -ht->nTableSize;
        HT_SET_DATA_ADDR(ht, new_data);
        memcpy(ht->arData, old_buckets, sizeof(Bucket) * ht->nNumUsed);// 复制原数组到新数组
        pefree(old_data, ht->u.flags & HASH_FLAG_PERSISTENT);// 释放原数组内存
        zend_hash_rehash(ht);// 重新索引
    } else {
        // 数组大小超出内存限制
        zend_error_noreturn(E_ERROR, "Possible integer overflow in memory allocation (%u * %zu + %zu)", ht->nTableSize * 2, sizeof(Bucket) + sizeof(uint32_t), sizeof(Bucket));
    }
}
```

#### HashTable分为packed_array和hash_array


packed array有以下约束和特性：
* key全是数字key
* key按插入顺序排列，并且是递增的
* 每一个key-value对应的存储位置都是确定的，都存储在buclet数组的第key个元素上
* packed array不需要索引数组（空间优化，并且存取也是直接操作bucekt数组，不需要借助索引数组，性能也有提升）
* nTableMask始终为-2

注意：PHP7会在packed array的空间效率以及时间效率优化与空间浪费之间做一个平衡，当空间浪费过多时（比如下标为，1，8，中奖浪费7个bucket空间），则会将packed array转化为hash array。转换函数为*zend_hash_packed_to_hash(ht)*;

packed_array不会用到映射表,如果h为数字,h的值就是arData的索引值,源码如下:


```
static zend_always_inline zval *zend_fetch_dimension_address_inner(HashTable *ht, const zval *dim, int dim_type, int type)
{
    zval *retval;
    zend_string *offset_key;
    zend_ulong hval;

try_again:
    if (EXPECTED(Z_TYPE_P(dim) == IS_LONG)) {
        hval = Z_LVAL_P(dim);
num_index://如果key是数字
        ZEND_HASH_INDEX_FIND(ht, hval, retval, num_undef);
        return retval;
```


```
#define ZEND_HASH_INDEX_FIND(_ht, _h, _ret, _not_found) do { 
        if (EXPECTED((_ht)->u.flags & HASH_FLAG_PACKED)) { 
            if (EXPECTED((zend_ulong)(_h) < (zend_ulong)(_ht)->nNumUsed)) { 
                _ret = &_ht->arData[_h].val; //h的值就是arData的下标idx
                if (UNEXPECTED(Z_TYPE_P(_ret) == IS_UNDEF)) { 
                    goto _not_found; 
                } 
            } else { 
                goto _not_found; 
            } 
        } else { 
            _ret = _zend_hash_index_find(_ht, _h); 
            if (UNEXPECTED(_ret == NULL)) { 
                goto _not_found; 
            } 
        } 
    } while (0)
```

最后做个实验来验证一下:
##### 1.packed_array.php

```
<?php
$memory_start=memory_get_usage();
$test=array();
for($i=0;$i<=200000;$i++)
{
	$test[$i]=1;
}
echo memory_get_usage() - $memory_start."\n";  //输出结果为 8392896
```

##### 2.hash_array.php


```
<?php
$memory_start=memory_get_usage();
$test=array();
for($i=200000;$i>=0;$i--)
{
	$test[$i]=1;
}
echo memory_get_usage() - $memory_start."\n";  //输出结果为 9437376
```
hash_array    占用了 9437376 byte,
packed_array占用了 8392896 byte,
大概节约了1M内存
所以能用packed_array尽量用packed_array,可以节省很多内存







