# Redis底层数据结构-Intset源码分析


# 简介
Intset是一个有序的，存储Integer类型数据的结构，当元素为64位有符号整数范围之内时，它是Redis数据结构中有序集合ZSET的底层实现，但是当元素个数超过一定数量时（默认为512），会转为hashtable进行存储，由配置项`set-max-intset-entries 512`决定。如果向有序集合中添加非整型变量，底层实现也会装欢为hashtable。



# 数据结构

intset结构如下：
```C

#define INTSET_ENC_INT16 (sizeof(int16_t))
#define INTSET_ENC_INT32： (sizeof(int32_t))
#define INTSET_ENC_INT64 (sizeof(int64_t))

typedef struct intset {
    uint32_t encoding;
    uint32_t length;
    int8_t contents[];
} intset;
```
各字段含义：
- encoding：编码类型，决定每个元素占用几个字节
    - INTSET_ENC_INT16：当元素大小位于`INT16_MIN`和`INT16_MAX`之间使用，每个元素占用两个字节。
    - INTSET_ENC_INT32：元素大小位于`INT16_MAX`和`INT32_MAX`之间或`INT32_MIN`到`INT16_MIN`之间，该编码方式占用四个字节。
    - INTSET_ENC_INT64：元素大小位于`INT32_MAX`到`INT64_MAX`或`INT64_MIN`到`INT32_MIN`之间，该编码方式每个元素占用八个字节。
- length：元素个数
- contents：存储具体元素的数组，从小到大排序，并且不包含任何重复项



当添加的新元素比集合里面所有元素类型都长时，集合需要整体进行upgrade，然后完成添加。








# 接口



## intsetFind

查找算法主要通过二分查找实现。
```C
/* Determine whether a value belongs to this set */
uint8_t intsetFind(intset *is, int64_t value) {
    //判断编码方式
    uint8_t valenc = _intsetValueEncoding(value);
    //编码方式如果当前intset的编码方式，直接返回0
    //否则调用intsetSearch函数进行查找
    return valenc &lt;= intrev32ifbe(is-&gt;encoding) &amp;&amp; intsetSearch(is,value,NULL);
}


static uint8_t intsetSearch(intset *is, int64_t value, uint32_t *pos) {
    int min = 0, max = intrev32ifbe(is-&gt;length)-1, mid = -1;
    int64_t cur = -1;

    /* The value can never be found when the set is empty */
    //intset如果为空，直接返回0
    if (intrev32ifbe(is-&gt;length) == 0) {
        if (pos) *pos = 0;
        return 0;
    } else {
        当前元素如果大于当前有序集合最大值或者小于最小值，直接返回0
        if (value &gt; _intsetGet(is,max)) {
            if (pos) *pos = intrev32ifbe(is-&gt;length);
            return 0;
        } else if (value &lt; _intsetGet(is,0)) {
            if (pos) *pos = 0;
            return 0;
        }
    }
    //由于集合有序，采用二分查找
    while(max &gt;= min) {
        mid = ((unsigned int)min &#43; (unsigned int)max) &gt;&gt; 1;
        cur = _intsetGet(is,mid);
        if (value &gt; cur) {
            min = mid&#43;1;
        } else if (value &lt; cur) {
            max = mid-1;
        } else {
            break;
        }
    }

    if (value == cur) {
        if (pos) *pos = mid;
        return 1;
    } else {
        if (pos) *pos = min;
        return 0;
    }
}
```

## intsetAdd

主要逻辑如下：
- 判断插入的值的编码格式，小于当前intset编码格式直接插入，大于则调用`intsetUpgradeAndAdd`函数进行升级
- 上述条件小于时，调用`intsetSearch`进行查找，如果插入的值已经存在，直接返回。
- 不存在则进行扩容操作，`intsetResize`将intset的长度加一。
- 如果插入的位置位于原来元素之间，使用`intsetMoveTail`进行元素移动，空出一个位置进行插入。
- 调用`_intsetSet`保存元素

```C
intset *intsetAdd(intset *is, int64_t value, uint8_t *success) {
    //获取添加元素的编码值
    uint8_t valenc = _intsetValueEncoding(value);
    uint32_t pos;
    if (success) *success = 1;

    
    if (valenc &gt; intrev32ifbe(is-&gt;encoding)) {
        //如果该元素编码大于当前intset编码。进行升级
        return intsetUpgradeAndAdd(is,value);
    } else {
        //否则先进行查重，如果已经存在该元素，直接返回（保证有序集合的唯一性）
        if (intsetSearch(is,value,&amp;pos)) {
            if (success) *success = 0;
            return is;
        }
        //扩容
        is = intsetResize(is,intrev32ifbe(is-&gt;length)&#43;1);
        //如果插入元素位置合法，将该位置往后挪
        if (pos &lt; intrev32ifbe(is-&gt;length)) intsetMoveTail(is,pos,pos&#43;1);
    }
    //保存元素
    _intsetSet(is,pos,value);
    //修改inset的长度，将其加1
    is-&gt;length = intrev32ifbe(intrev32ifbe(is-&gt;length)&#43;1);
    return is;
}

static intset *intsetUpgradeAndAdd(intset *is, int64_t value) {
    uint8_t curenc = intrev32ifbe(is-&gt;encoding);
    uint8_t newenc = _intsetValueEncoding(value);
    int length = intrev32ifbe(is-&gt;length);
    int prepend = value &lt; 0 ? 1 : 0;

    /* First set new encoding and resize */
    is-&gt;encoding = intrev32ifbe(newenc);
    is = intsetResize(is,intrev32ifbe(is-&gt;length)&#43;1);

    /* Upgrade back-to-front so we don&#39;t overwrite values.
     * Note that the &#34;prepend&#34; variable is used to make sure we have an empty
     * space at either the beginning or the end of the intset. */
    while(length--)
        _intsetSet(is,length&#43;prepend,_intsetGetEncoded(is,length,curenc));

    /* Set the value at the beginning or the end. */
    if (prepend)
        _intsetSet(is,0,value);
    else
        _intsetSet(is,intrev32ifbe(is-&gt;length),value);
    is-&gt;length = intrev32ifbe(intrev32ifbe(is-&gt;length)&#43;1);
    return is;
}
```

## intsetRemove

主要逻辑如下：
- 判断新插入值的编码是否小于等于当前intset编码。
- 该值是否在intset中存在，如果存在，将intset的值从pos &#43; 1的位置移动到pos位置。
- 更新长度字段。

```C
intset *intsetRemove(intset *is, int64_t value, int *success) {
    //获取待删除元素的编码
    uint8_t valenc = _intsetValueEncoding(value);
    uint32_t pos;
    if (success) *success = 0;
    //如果编码小于等于当前intset编码并且查找到该元素，则进行删除操作
    if (valenc &lt;= intrev32ifbe(is-&gt;encoding) &amp;&amp; intsetSearch(is,value,&amp;pos)) {
        //获取intset长度
        uint32_t len = intrev32ifbe(is-&gt;length);

        /* We know we can delete */
        if (success) *success = 1;

        //调用intsetMoveTail对待删除元素进行覆盖
        if (pos &lt; (len-1)) intsetMoveTail(is,pos&#43;1,pos);
        is = intsetResize(is,len-1);
        is-&gt;length = intrev32ifbe(len-1);
    }
    return is;
}
```

# 总结
本文主要对Redis中集合的底层数据结构intset结构及其基本操作函数做了简要分析，该结构主要使用场景为集合元素全为整型且元素数量不多（默认小于512）。

---

> Author:   
> URL: http://localhost:1313/posts/%E4%B8%AD%E9%97%B4%E4%BB%B6/redis/redis-intset/  

