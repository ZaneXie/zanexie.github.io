---
title: PHP7之HashTable
date: 2015-12-09 20:04:16
tags:
---

HashTable是PHP内核里十分重要的数据结构，承载着PHP大部分的语法特性，例如：函数表，类的属性、方法等。

为了提高性能，PHP7重新实现了HashTable，相较之前的版本在内存和CPU的使用上均有了很大的提升。[Nikic](https://nikic.github.io/aboutMe.html)一年前（2014年12月）发布在github.io上的文章[PHP's new hashtable implementation](https://nikic.github.io/2014/12/22/PHPs-new-hashtable-implementation.html) ，详细解释了新的实现，并说明了为何比之前的实现效率更高。不过文章发布时PHP7还在开发阶段，最终release的时候HashTable做了一些细微的调整，与Nikic的文章略有差别，在下文中会做详细介绍。

## PHP5的HashTable
PHP5的HashTable，为解决Hash冲突使用了链表，大致的实现如下图所示：

{% asset_img basic_hashtable.svg basic_hashtable %}

更详细的描述可以在PHP Internals Books的[Basic Structure](http://www.phpinternalsbook.com/hashtables/basic_structure.html)章节找到。

## PHP7的HashTable

PHP5的HashTable使用链表解决Hash冲突，虽然实现了常数时间的增删查改，但是产生了一些额外的开销，比如：每个Bucket需要单独申请内存（allocation)，不仅耗时而且为存储指针需要额外的空间开销；为实现数组有序而使用的双向链表会占用额外的空间…… 为了提高HashTable的性能，PHP7重新定义了PHP最基础的数据结构zval，并以此重新设计了HashTable。

### 新的zval
先来看看新的zend_value，

```c
typedef union _zend_value {
    zend_long         lval;                /* long value */
    double            dval;                /* double value */
    zend_refcounted  *counted;
    zend_string      *str;
    zend_array       *arr;
    zend_object      *obj;
    zend_resource    *res;
    zend_reference   *ref;
    zend_ast_ref     *ast;
    zval             *zv;
    void             *ptr;
    zend_class_entry *ce;
    zend_function    *func;
    struct {
        uint32_t w1;
        uint32_t w2;
    } ww;
} zend_value;
```

zend_value将简单数值（long和double）嵌入在了zend_value中，其他类型均使用一个指针指向实际的地址。需要注意的是，这是一个union，也就是说这里定义的所有元素共享一块内存，在64位机器上，long、double、指针均是8字节，所以这里一共占用8字节。 接下来就是zval

```c
struct _zval_struct {
    zend_value        value;            /* value */
    union {
        struct {
            ZEND_ENDIAN_LOHI_4(
                zend_uchar    type,            /* active type */
                zend_uchar    type_flags,
                zend_uchar    const_flags,
                zend_uchar    reserved)        /* call info for EX(This) */
        } v;
        uint32_t type_info;
    } u1;
    union {
        uint32_t     var_flags;
        uint32_t     next;                 /* hash collision chain */
        uint32_t     cache_slot;           /* literal cache slot */
        uint32_t     lineno;               /* line number (for ast nodes) */
        uint32_t     num_args;             /* arguments number for EX(This) */
        uint32_t     fe_pos;               /* foreach position */
        uint32_t     fe_iter_idx;          /* foreach iterator index */
    } u2;
};
```

zval里首先是一个zend_value，用于存放变量的值。接下来是u1，这也是一个union，v和type_info共享4字节的内存，用于保存zval中存储的值的类型（long/double/string/object等）。接下来是一个trick。首先需要知道，在64位机器上，将struct按8字节（64bit）对齐可以获得更好的执行效率（<a href="https://en.wikipedia.org/wiki/Data_structure_alignment">Data structure alignment</a>）。在zval中，必须的两项value与type分别占据了8字节与4字节，虽然只使用了12字节，但是在编译时为了更好的性能，会按16字节对齐，这样后面就空出了4字节。为了更好的利用这部分空间，将最后4字节定义为union u2，用于在不同场合发挥功能，例如，其中的u2.next用于在HashTable里寻找下一个元素。（类似的trick在代码里随处可见，读的时候想最多的是“卧槽，太牛逼了”。）

### HashTable
终于，我们的主角登场了。
```c
typedef struct _Bucket {
    zval              val;
    zend_ulong        h;   /* hash value (or numeric index)   */
    zend_string      *key; /* string key or NULL for numerics */
} Bucket;

typedef struct _zend_array HashTable;

struct _zend_array {
    zend_refcounted_h gc; 
    union {
        struct {
            ZEND_ENDIAN_LOHI_4(
                zend_uchar    flags,
                zend_uchar    nApplyCount,
                zend_uchar    nIteratorsCount,
                zend_uchar    reserve)
        } v;
        uint32_t flags; 
    } u;
    uint32_t          nTableMask; //Table Mask, 始终等于-nTableSize
    Bucket           *arData;     //实际的数据存放位置
    uint32_t          nNumUsed;   //已使用的槽位数
    uint32_t          nNumOfElements; //已有的元素个数，注意这个值可能会小于nNumUsed
    uint32_t          nTableSize;  //HashTable的大小，始终为2的指数（8,16,32,64...）。最小为8，最大值根据机器不同而不同
    uint32_t          nInternalPointer;
    zend_long         nNextFreeElement;
    dtor_func_t       pDestructor;
};
```

这是全新的HashTable，与之前实现最大的区别是所有的Bucket作为一个数组，放在了同一块内存上（PHP5中的Bucket分别申请，并用链表连接在一起)。文章开头提到的PHP7正式版与Nikic文章中的主要区别就是这了，为了更好的理解HashTable的实现，先看看原来的结构：

```c
typedef struct _HashTable {
    uint32_t          nTableSize;
    uint32_t          nTableMask;
    uint32_t          nNumUsed;
    uint32_t          nNumOfElements;
    zend_long         nNextFreeElement;
    Bucket           *arData;        //实际的数据
    uint32_t         *arHash;        //哈希表。特定哈希值对应的数组下标。
    dtor_func_t       pDestructor;
    uint32_t          nInternalPointer;
    union {
        struct {
            ZEND_ENDIAN_LOHI_3(
                zend_uchar    flags,
                zend_uchar    nApplyCount,
                uint16_t      reserve)
        } v;
        uint32_t flags;
    } u;
} HashTable;
```

存储中，最关键的两个是两个指针*arData和*arHash。其中，arData是Bucket的实际存储位置，在HashTable初始化的时候，会分配一块连续的能连续存放nTableSize个Bucket的内存，因此在使用时可以将其当作数组访问：arData[0], arData[1]……；arHash是一个nTableSize大小的数组，元素的key在hash之后落在0~(nTableSize-1)之间，这个数组是arData的索引，用于根据hash值迅速找到其对应的元素。

下面通过实例从增与查两个最基本的操作来说明具体的实现。为了描述方便，下面使用了简化的算法，实际的实现要复杂不少，但本质是一致的。

假定有一个空数组a = []。初始化好后，主要元素的信息如下：
```c 
nTableSize=8
nNumUsed=0
nNumOfElements=0
arHash=[-1,-1,-1....-1] （8个）
arData=[Bucket(NOT_INITIALIZED), Bucket(NOT_INITIALIZED)....]（8个空位）
```

**增（Insert）**

比如执行代码a['x'] = 1。首先，计算'x'的hash值，并mod nTableSize，结果记为nHashIndex，根据结果在arHash数组中对应位置查找结果，如果值为-1，表示当前hash值无冲突，将value=1写入arData[nNumUsed]位置，并将该位置记录在arHash表中；如果值不为-1，表示hash值有冲突，需要解决，详细的解决方案见下文。伪代码如下：
```c
int nHashIndex = hash(key) % nTableSize;
int nDataIndex = arHash[nHashIndex];
if(nDataIndex != -1){
    //解决冲突
}
arData[nNumUsed]=Bucket(value);
arHash[nHashIndex]=nNumUsed;
nNumUsed++;
nNumOfElements++;
```

a['x'] = 1执行结束后，
```c
nTableSize=8
nNumUsed=1
nNumOfElements=1
arHash=[0,-1,-1....-1] （一个0与7个-1，假定hash('x')=0）
arData=[Bucket（value = 1), Bucket(NOT_INITIALIZED)....]（7个空位）```
如此，一次插入a['y']=2; a['z']=3;..
```c
nTableSize=8
nNumUsed=3
nNumOfElements=3
arHash=[0,1,2....-1] （0,1,2与5个-1，假定hash('x'/'y'/'z')分别等于0,1,2）
arData=[Bucket（value = 1), Bucket(value = 2), Bucket(value = 3)....]（5个空位）
```

**查（Find**

上面三个元素插入之后，如果要取出来，怎么操作呢？比如，要知道a['y']对应的值，伪代码如下：
```c
int nHashIndex = hash(key) % nTableSize;
int nDataIndex = arHash[nHashIndex];
if (nDataIndex != -1){
    Bucket *target = &amp;arData[nDataIndex];
    if (target-&gt;key == key){
        return target;
    }else{
      //未找到或者遇到Hash冲突
    }
}
 
return NULL; //NOT FOUND

```
对应上一步的结果，可以算出nHashIndex=1，从arHash<a href="/wp-content/uploads/2015/12/basic_hashtable.svg">1</a>中取出1，因此a['y] = arData<a href="https://nikic.github.io/2014/12/22/PHPs-new-hashtable-implementation.html">1</a> = Bucket(value = 2)。如此，完成了一次查找。

**Hash冲突** 

在插入时，如果遇到Hash冲突（即两个key的hash值结果一致时），上面提到的zval中空余的4字节就发挥作用了：指向同hash值的下一个元素的位置。现在来完成上面的伪代码，只需增加一行：
```c
int nHashIndex = hash(key) % nTableSize;
int nDataIndex = arHash[nHashIndex];
if(nDataIndex != -1){
    value.u2.next=nDataIndex;  //value是一个zval
}
arData[nNumUsed]=Bucket(value); 
arHash[nHashIndex]=nNumUsed;
nNumUsed++;
nNumOfElements++;
```

在查找时，当target-&gt;key与带查找的key不相同时，可以根据target-&gt;val.u2.next来找到下一个nDataIndex，上面的伪代码加上冲突解决之后：

```c
int nHashIndex = hash(key) % nTableSize;
int nDataIndex = arHash[nHashIndex];
while (nDataIndex != -1){
    Bucket *target = &amp;arData[nDataIndex];
    if (target-&gt;key == key){
        return target;
    }else{
        nDataIndex = target-&gt;value.u2.next;
    }
}
 
return NULL; //未找到
```
**删除**

在上面的代码中，能看到HashTable里有nNumUsed和nNumOfElements，这两个值从代码里看，是同时增长，为什么需要分开记录呢？其实，这是为了让删除这个操作更容易。 新的HashTable使用了连续的内存来存放数据，如果每删除一个元素，就将其后的元素全部往前移动一个位置，在性能上是惨不忍睹的，也是完全没有必要的。实际中，找到key对应的元素后，将其标记为UNDEF，并将nNumOfElements减一就可以了，并不需要真实的删除。比如还是上面的array：$a=['x' =&gt; 1, 'y'=&gt;2, 'z'=&gt;3], 如果执行unset($a['y'])，数据将会变成下面这样：

```c
nTableSize=8
nNumUsed=3
nNumOfElements=2
arHash=[0,-1,2....-1] （0,2与6个-1，假定hash('x'/'y'/'z')分别等于0,1,2）
arData=[Bucket（value = 1), Bucket(UNDEF), Bucket(value = 3)....]（5个空位）
```
这样就完成了元素的删除。

**顺序**

PHP的数组是有序的，先插入的在前而后插入的靠后，比如 $a=[3,4]就和$a[4,3]不一样。在PHP5为了保持数组有序，使用了双向链表，而在PHP7中，更先进的设计可以让系统不再存储这样的链表。从上面的伪代码中可以看到，后插入的数据，总是放在了arData[nNumUsed]中，即：后插入的元素总是在arData靠后的位置，遍历时按arData的顺序进行ok了，不再需要额外的指针。
### HashTable的实现
上面的伪代码只是最基础的思想，在PHP7的代码实现中，使用了更有效的方式来提高效率。

**与Nikic文章中的区别**

在前面提到过，正式版的PHP7与Nikic文章中有一些区别，具体来说，正式版中移除了uint32_t *arHash指针。nTablesMask从nTableSize-1，变成了-nTableSize。 有了上面的算法，来理解这两个改变就比较容易了。 正式的PHP7中，arHash和arData共享了同一段内存。引用一段zend_types.h中的注释：
```c
/*
 * HashTable Data Layout
 * =====================
 *
 *                 +=============================+
 *                 | HT_HASH(ht, ht->nTableMask) |
 *                 | ...                         |
 *                 | HT_HASH(ht, -1)             |
 *                 +-----------------------------+
 * ht->arData ---> | Bucket[0]                   |
 *                 | ...                         |
 *                 | Bucket[ht->nTableSize-1]    |
 *                 +=============================+
 */
```
在malloc时，同时申请了arHash和arData共需的内存空间。并将arData的指针指向这段内存中间的地址，指针之前的位置放置arHash，之后的是arData。

nTableMask，之前值为nTableSize-1的原因是方便取模：在nTableSize为2的指数时，总有 h &amp; (nTableSize-1) = h % nTableSize ，可以用 &amp; 替代耗时的 %。例如当nTableSize = 8时，nTableSize - 1 = 7 ，表示为二进制是 000...0111，将h与这个值做 &amp; 操作，结果与 h%8 相同。

新的值为-nTableSize的原因也是方便取模：因为新的arHash放在了指针arData之前，所以需要取负数的模，在nTableSize为2的指数时，h | (-nTableSize) = - nTableSize + ( h % nTableSize) ，如上图，相当于是arHash的第 h% nTableSize 个位置。例如当nTableSize = 8时， -nTableSize = -8，表示为二进制为111111....1000（<a href="https://en.wikipedia.org/wiki/Two%27s_complement">补码</a>），与h做|预算，结果是111111....1xyz = -8 + xyz，( xyz = h % 8 )，巧妙的通过|算的了下标。

这两个指针共享同一段内存，在初始化时可以减少一次malloc，销毁时减少一次free，同时还可以节省保存arHash指针的空间。而通过上面的算法，实现这些不需要任何额外的开销。

**Packed HashTable**

PHP7的引入了Packed HashTable，在一些特定的情况下，Packed HashTable能达到与C语言的数组相近的性能。 先回到之前zval和Bucket的定义，可以看到long和double是直接嵌入到zval中的，zval是直接嵌入到Bucket中的。这意味着每个Bucket中就有一个long或者double，也就是说，arData中，默认是有long/double的位置的。所以，当使用一个纯粹的数字数组时，比如 $a = [3, 4, 5, 6, 7], $b = [3.3, 4.3]，只需要申请一次arData的空间，然后用下标做索引，就实现了HashTable。此时访问这个数组，将不需要额外的查找，而创建时也只花费了一次malloc的时间，性能已接近C的数组。

**arHash的类型**

uint32_t *arHash。一般我们为了指向一个元素，会保存它的指针，在64位系统中，一个指针占据8个字节。而PHP7并没有将其直接指向元素，而是作为一个下标，与arData配合得到元素的位置（HashTable的最大元素个数为 32位机 0x04000000个， 64位机0x80000000个，均未超过uint32的范围)。这样做最直观的好处是省内存，同时，还有一个容易忽略的地方：zval.u2.next大小是32位。此处如果直接保存指针，zval.u2.next就无法保存下一个元素的位置了，而如果把zval.u2.next定义为64位，zval又会占据更多的空间。

**HashTable 容量**

HashTable的大小为 8 ~ 0x04000000（32位）/ 0x80000000（64位），且只允许为2的指数。定义在zend_types.h，HT_MIN_SIZE与HT_MAX_SIZE。

初始化时，如果指定的容量小于8，将会被强制设定为8，如果大于8，将会设置为比该值大的最小的2的指数，比如，如果设置为12，将得到一个大小为16的HashTable。如果大于HT_MAX_SIZE，触发E_ERROR， “Possible integer overflow in memory allocation”。

使用时，当HashTable中的元素超过nTableSize时，会将其扩大到原来容量的两倍，将原来的数据拷贝并重新计算arHash。

**Hash算法**

PHP7使用了Daniel J. Bernstein的DJBX33A算法，核心思想是 hash (i) = hash (i - 1) * 33 + str[i]，详细的实现在zend_string.c中。
