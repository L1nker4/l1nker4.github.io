# Redis底层数据结构-Quicklist源码分析


# 简介
quicklist是Redis 3.2中引入的数据结构，其本质是一个双向链表，链表的节点类型是ziplist，当ziplist节点过多，quicklist会退化成双向链表，以提高效率。

# 数据结构

## quicklistNode
该结构为节点类型。
- prev：指向前驱节点
- next：指向后继节点
- zl：指向对应的压缩列表
- sz：压缩列表的大小
- encoding：采用的编码方式，1为原生，2代表使用LZF进行压缩
- container：为节点指向的容器类型，1代表none，2代表ziplist存储数据
- recompress：代表这个节点是否是压缩节点，如果是，则使用压缩节点前先进行解压缩，使用后重新压缩
- attempted_compress：测试时使用
```C
typedef struct quicklistNode {
    struct quicklistNode *prev;
    struct quicklistNode *next;
    unsigned char *zl;
    unsigned int sz;             /* ziplist size in bytes */
    unsigned int count : 16;     /* count of items in ziplist */
    unsigned int encoding : 2;   /* RAW==1 or LZF==2 */
    unsigned int container : 2;  /* NONE==1 or ZIPLIST==2 */
    unsigned int recompress : 1; /* was this node previous compressed? */
    unsigned int attempted_compress : 1; /* node can&#39;t compress; too small */
    unsigned int extra : 10; /* more bits to steal for future usage */
} quicklistNode;
```

## quicklistLZF
如果使用LZF算法进行压缩，节点指向的结构为quicklistLZF，其中sz表示compressed数组所占用字节大小。
```C
typedef struct quicklistLZF {
    unsigned int sz; /* LZF size in bytes*/
    char compressed[];
} quicklistLZF;
```
## quicklist
- head：指向头节点
- tail：指向尾节点
- count：quicklist中压缩列表的entry总数
- len：节点个数
- fill：每个节点的ziplist长度，可以通过参数`list-max-ziplist-size`配置节点所占内存大小
- compress：该值表示两端各有compress个节点不压缩
```C
typedef struct quicklist {
    quicklistNode *head;
    quicklistNode *tail;
    unsigned long count;        /* total count of all entries in all ziplists */
    unsigned long len;          /* number of quicklistNodes */
    int fill : 16;              /* fill factor for individual nodes */
    unsigned int compress : 16; /* depth of end nodes not to compress;0=off */
} quicklist;
```

## quicklistEntry
- quicklist：指向当前元素所在的quicklist
- node：指向当前元素所在的节点
- zi：当前元素所在的ziplist
- value：当前节点的字符串内容
- longval：当前节点的整型值
- sz：节点的大小
- offset：该节点相对于整个ziplist的偏移量，即相当于第几个entry
```C
typedef struct quicklistEntry {
    const quicklist *quicklist;
    quicklistNode *node;
    unsigned char *zi;
    unsigned char *value;
    long long longval;
    unsigned int sz;
    int offset;
} quicklistEntry;
```

## quicklistIter
用于quicklist遍历的迭代器。
- quicklist：指向对应的quicklist
- current：指向元素所在的节点
- zi：指向元素所在的ziplist
- offset：表示节点在ziplist中的偏移量
- direction：表示迭代器的方向
```C
typedef struct quicklistIter {
    const quicklist *quicklist;
    quicklistNode *current;
    unsigned char *zi;
    long offset; /* offset in current ziplist */
    int direction;
} quicklistIter;
```

# 接口

## quicklistPushHead

该接口在头部插入元素。传入参数分别是待插入的quicklist，插入的数据value，大小sz。返回指为0代表没有新建head节点，1代表新建了head节点
基本逻辑如下：
- 查看quicklist的head节点是否允许插入，可以就利用ziplist的接口进行插入。
- 否则新建head节点进行插入。
```C
int quicklistPushHead(quicklist *quicklist, void *value, size_t sz) {
    quicklistNode *orig_head = quicklist-&gt;head;
    //检查头节点是否允许插入
    if (likely(
            _quicklistNodeAllowInsert(quicklist-&gt;head, quicklist-&gt;fill, sz))) {
            //使用头节点中的ziplist接口进行插入
        quicklist-&gt;head-&gt;zl =
            ziplistPush(quicklist-&gt;head-&gt;zl, value, sz, ZIPLIST_HEAD);
        quicklistNodeUpdateSz(quicklist-&gt;head);
    } else {
        //创建新节点进行插入
        quicklistNode *node = quicklistCreateNode();
        node-&gt;zl = ziplistPush(ziplistNew(), value, sz, ZIPLIST_HEAD);

        quicklistNodeUpdateSz(node);
        _quicklistInsertNodeBefore(quicklist, quicklist-&gt;head, node);
    }
    //更新对应数据
    quicklist-&gt;count&#43;&#43;;
    quicklist-&gt;head-&gt;count&#43;&#43;;
    return (orig_head != quicklist-&gt;head);
}
```

## quicklistReplaceAtIndex
更改元素需要传入index，基本逻辑是：
- 删除原有元素
- 插入新元素
```C
int quicklistReplaceAtIndex(quicklist *quicklist, long index, void *data,
                            int sz) {
    quicklistEntry entry;
    //检查index位置的元素是否存在
    if (likely(quicklistIndex(quicklist, index, &amp;entry))) {
        //删除
        entry.node-&gt;zl = ziplistDelete(entry.node-&gt;zl, &amp;entry.zi);
        //添加
        entry.node-&gt;zl = ziplistInsert(entry.node-&gt;zl, entry.zi, data, sz);
        //更新sz字段
        quicklistNodeUpdateSz(entry.node);
        quicklistCompress(quicklist, entry.node);
        return 1;
    } else {
        return 0;
    }
}
```

## quicklistIndex
该方法主要通过元素的下标查找对应元素，找到index对应数据所在的节点，再调用ziplist的`ziplistGet`得到Index对应的数据。
```C

//idx为需要查找的下标, 结果写入entry，返回0代表没找到，1代表找到
int quicklistIndex(const quicklist *quicklist, const long long idx,
                   quicklistEntry *entry) {
    quicklistNode *n;
    unsigned long long accum = 0;
    unsigned long long index;
    //idx为负值，表示从尾部向头部的偏移量
    int forward = idx &lt; 0 ? 0 : 1; /* &lt; 0 -&gt; reverse, 0&#43; -&gt; forward */
    //初始化entry
    initEntry(entry);
    entry-&gt;quicklist = quicklist;

    if (!forward) {
        //idx小于0，将idx设置为(-idx) - 1，从尾部查找
        index = (-idx) - 1;
        n = quicklist-&gt;tail;
    } else {
        index = idx;
        n = quicklist-&gt;head;
    }

    if (index &gt;= quicklist-&gt;count)
        return 0;
    //遍历Node节点，找到对应的index元素
    while (likely(n)) {
        if ((accum &#43; n-&gt;count) &gt; index) {
            break;
        } else {
            D(&#34;Skipping over (%p) %u at accum %lld&#34;, (void *)n, n-&gt;count,
              accum);
            accum &#43;= n-&gt;count;
            n = forward ? n-&gt;next : n-&gt;prev;
        }
    }

    if (!n)
        return 0;

    D(&#34;Found node: %p at accum %llu, idx %llu, sub&#43; %llu, sub- %llu&#34;, (void *)n,
      accum, index, index - accum, (-index) - 1 &#43; accum);
    //计算index所在的ziplist的偏移量
    entry-&gt;node = n;
    if (forward) {
        /* forward = normal head-to-tail offset. */
        entry-&gt;offset = index - accum;
    } else {
        /* reverse = need negative offset for tail-to-head, so undo
         * the result of the original if (index &lt; 0) above. */
        entry-&gt;offset = (-index) - 1 &#43; accum;
    }

    quicklistDecompressNodeForUse(entry-&gt;node);
    entry-&gt;zi = ziplistIndex(entry-&gt;node-&gt;zl, entry-&gt;offset);
    //利用ziplistGet获取元素
    ziplistGet(entry-&gt;zi, &amp;entry-&gt;value, &amp;entry-&gt;sz, &amp;entry-&gt;longval);
    /* The caller will use our result, so we don&#39;t re-compress here.
     * The caller can recompress or delete the node as needed. */
    return 1;
}
```

# 总结
本文主要介绍了Quicklist的基本数据结构，并对该结构的基本增删改查接口做了简要分析。

---

> Author:   
> URL: http://localhost:1313/posts/%E4%B8%AD%E9%97%B4%E4%BB%B6/redis/redis-quicklist/  

