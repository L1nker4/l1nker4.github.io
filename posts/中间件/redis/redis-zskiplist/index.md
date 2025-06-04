# Redis底层数据结构-SkipList源码分析




# 简介

跳表(SkipList)通过对有序链表添加多级索引，从而实现类似于二分查找效果的有序链表，它的插入/删除/搜索的平均时间复杂度为`O(log n)`，该数据结构可以用来代替平衡树以提高效率。其基本结构如 下图所示：

![跳表基本结构](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/redis/skiplist/skiplist.png)



如果此时查找51的节点，步骤基本如下：

1. 从第二层开始查找，1比51小，向后比较
2. 21比51小，21后面为NULL，下降到第一层的21先后比较
3. 第一层中21的next节点为41，41比51小，41的next节点61比51大，下降到第0层比较
4. 41的next节点为51，查找完成。







# 数据结构

zskiplistNode：

- ele：存储SDS类型的数据
- score：排序用的分值
- backward：后退指针，指向当前节点最底层的前驱节点，第一个指向NULL
- level：数组，它的长度在生成时随机生成一个1 ~ 64的值，值越大出现概率越低
  - forward：指向本层的下一个节点，尾节点指向NULL
  - span：指向的节点与本节点之间的元素个数

zskiplist：

- header：指向跳表的头节点
- tail：指向跳表的尾节点
- length：跳表的长度
- level：跳表的高度

```C
/* ZSETs use a specialized version of Skiplists */
typedef struct zskiplistNode {
    sds ele;
    double score;
    struct zskiplistNode *backward;
    struct zskiplistLevel {
        struct zskiplistNode *forward;
        unsigned long span;
    } level[];
} zskiplistNode;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;
    int level;
} zskiplist;

typedef struct zset {
    dict *dict;
    zskiplist *zsl;
} zset;
```



# 接口



## zslCreate

### zslCreateNode

```C
/* Create a skiplist node with the specified number of levels.
 * The SDS string &#39;ele&#39; is referenced by the node after the call. */
zskiplistNode *zslCreateNode(int level, double score, sds ele) {
    //申请内存空间
    zskiplistNode *zn =
        zmalloc(sizeof(*zn)&#43;level*sizeof(struct zskiplistLevel));
    //初始化
    zn-&gt;score = score;
    zn-&gt;ele = ele;
    return zn;
}
```



### zslCreate

```C
/* Create a new skiplist. */
zskiplist *zslCreate(void) {
    int j;
    //指向跳表的指针
    zskiplist *zsl;
	//申请内存空间
    zsl = zmalloc(sizeof(*zsl));
    //设置默认值
    zsl-&gt;level = 1;
    zsl-&gt;length = 0;
    //创建头节点
    zsl-&gt;header = zslCreateNode(ZSKIPLIST_MAXLEVEL,0,NULL);
    //将头节点的level数组的forward设置为NULL，span设置为0
    for (j = 0; j &lt; ZSKIPLIST_MAXLEVEL; j&#43;&#43;) {
        zsl-&gt;header-&gt;level[j].forward = NULL;
        zsl-&gt;header-&gt;level[j].span = 0;
    }
    //设置头尾节点
    zsl-&gt;header-&gt;backward = NULL;
    zsl-&gt;tail = NULL;
    return zsl;
}
```





## zslRandomLevel

level最小值为1，最大值为64，该方法随机生成1 ~ 64的值。

```C
#define ZSKIPLIST_MAXLEVEL 64
#define ZSKIPLIST_P 0.25
int zslRandomLevel(void) {
    int level = 1;
    //生成随机值，取低16位为x，当x &lt; 0.25 * 0xFFFF时，level自增1
    while ((random()&amp;0xFFFF) &lt; (ZSKIPLIST_P * 0xFFFF))
        level &#43;= 1;
    return (level&lt;ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
}
```



## zslInsert

插入逻辑主要如下：

- 查找要插入的位置
- 调整跳表高度
- 插入节点
- 调整backward

```C
zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele) {
    //保存每一层需要更新的节点
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    //保存从header到update[i]节点的步长
    unsigned int rank[ZSKIPLIST_MAXLEVEL];
    int i, level;

    serverAssert(!isnan(score));
    //查找要插入的位置
    x = zsl-&gt;header;
    for (i = zsl-&gt;level-1; i &gt;= 0; i--) {
        /* store rank that is crossed to reach the insert position */
        rank[i] = i == (zsl-&gt;level-1) ? 0 : rank[i&#43;1];
        while (x-&gt;level[i].forward &amp;&amp;
                (x-&gt;level[i].forward-&gt;score &lt; score ||
                    (x-&gt;level[i].forward-&gt;score == score &amp;&amp;
                    sdscmp(x-&gt;level[i].forward-&gt;ele,ele) &lt; 0)))
        {
            rank[i] &#43;= x-&gt;level[i].span;
            x = x-&gt;level[i].forward;
        }
        update[i] = x;
    }
    //获取随机层数
    level = zslRandomLevel();
    //如果插入的层数大于最高层，设置rank和update数组
    if (level &gt; zsl-&gt;level) {
        for (i = zsl-&gt;level; i &lt; level; i&#43;&#43;) {
            rank[i] = 0;
            update[i] = zsl-&gt;header;
            update[i]-&gt;level[i].span = zsl-&gt;length;
        }
        zsl-&gt;level = level;
    }
    //创建节点
    x = zslCreateNode(level,score,ele);
    for (i = 0; i &lt; level; i&#43;&#43;) {
        //更新指向
        x-&gt;level[i].forward = update[i]-&gt;level[i].forward;
        update[i]-&gt;level[i].forward = x;

        //更新span
        x-&gt;level[i].span = update[i]-&gt;level[i].span - (rank[0] - rank[i]);
        update[i]-&gt;level[i].span = (rank[0] - rank[i]) &#43; 1;
    }

    /* increment span for untouched levels */
    for (i = level; i &lt; zsl-&gt;level; i&#43;&#43;) {
        update[i]-&gt;level[i].span&#43;&#43;;
    }
	//调整backward指针
    x-&gt;backward = (update[0] == zsl-&gt;header) ? NULL : update[0];
    if (x-&gt;level[0].forward)
        x-&gt;level[0].forward-&gt;backward = x;
    else
        zsl-&gt;tail = x;
    zsl-&gt;length&#43;&#43;;
    return x;
}
```



## zslDelete

删除逻辑主要如下：

- 查找需要删除的节点
- 设置span和forward

```C
// 删除指定的节点，节点是否删除取决于 node是否为 NULL，如果为 NULL的话就删除，否则将值存到 *node中
int zslDelete(zskiplist *zsl, double score, sds ele, zskiplistNode **node) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    int i;

    x = zsl-&gt;header;
    // 定位到每一层需要删除的位置
    for (i = zsl-&gt;level-1; i &gt;= 0; i--) {
        while (x-&gt;level[i].forward &amp;&amp;
                (x-&gt;level[i].forward-&gt;score &lt; score ||
                    (x-&gt;level[i].forward-&gt;score == score &amp;&amp;
                     sdscmp(x-&gt;level[i].forward-&gt;ele,ele) &lt; 0)))
        {
            x = x-&gt;level[i].forward;
        }
        update[i] = x;
    }
   
    // 在第一层(也就是完整的链表)中找到要删除的节点位置
    x = x-&gt;level[0].forward;
    // 完全相同才进行删除
    if (x &amp;&amp; score == x-&gt;score &amp;&amp; sdscmp(x-&gt;ele,ele) == 0) {
        zslDeleteNode(zsl, x, update);
        if (!node)
            zslFreeNode(x);
        else
            *node = x;
        return 1;
    }
    // 遍历结束也没找到
    return 0;
}

// 删除节点
void zslDeleteNode(zskiplist *zsl, zskiplistNode *x, zskiplistNode **update) {
    int i;
    for (i = 0; i &lt; zsl-&gt;level; i&#43;&#43;) {
        // 更新 span和前驱指针
        if (update[i]-&gt;level[i].forward == x) {
            update[i]-&gt;level[i].span &#43;= x-&gt;level[i].span - 1;
            update[i]-&gt;level[i].forward = x-&gt;level[i].forward;
        } else {
            // 不相等说明这些节点都在要删除的节点之前，跨度 span应该减 1
            update[i]-&gt;level[i].span -= 1;
        }
    }
    if (x-&gt;level[0].forward) {
        // 更新前驱指针，相当于更新双向链表的 prev指向
        x-&gt;level[0].forward-&gt;backward = x-&gt;backward;
    } else {
        // 说明要删除的节点是最后一个节点，更新尾节点
        zsl-&gt;tail = x-&gt;backward;
    }
    // 对于最上面都是空的层，应该排除，从第一个有数据的层开始算作有效
    while(zsl-&gt;level &gt; 1 &amp;&amp; zsl-&gt;header-&gt;level[zsl-&gt;level-1].forward == NULL)
        zsl-&gt;level--;
    zsl-&gt;length--;
}
```



## zslFree

删除跳表逻辑：

- 释放头节点
- 从头节点的第0层开始，通过forward向后遍历，逐个释放节点的内存空间
- 最后释放跳表指针空间

```C
/* Free a whole skiplist. */
void zslFree(zskiplist *zsl) {
    zskiplistNode *node = zsl-&gt;header-&gt;level[0].forward, *next;

    zfree(zsl-&gt;header);
    while(node) {
        next = node-&gt;level[0].forward;
        zslFreeNode(node);
        node = next;
    }
    zfree(zsl);
}

//释放节点，首先释放SDS的空间，再释放节点空间
void zslFreeNode(zskiplistNode *node) {
    sdsfree(node-&gt;ele);
    zfree(node);
}
```



## zslUpdateScore

更新节点的排序分数基本逻辑如下：

- 找到对应需要更新的节点

```C
zskiplistNode *zslUpdateScore(zskiplist *zsl, double curscore, sds ele, double newscore) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    int i;

    //找到对应需要更新的节点
    x = zsl-&gt;header;
    for (i = zsl-&gt;level-1; i &gt;= 0; i--) {
        while (x-&gt;level[i].forward &amp;&amp;
                (x-&gt;level[i].forward-&gt;score &lt; curscore ||
                    (x-&gt;level[i].forward-&gt;score == curscore &amp;&amp;
                     sdscmp(x-&gt;level[i].forward-&gt;ele,ele) &lt; 0)))
        {
            x = x-&gt;level[i].forward;
        }
        update[i] = x;
    }

    /* Jump to our element: note that this function assumes that the
     * element with the matching score exists. */
    x = x-&gt;level[0].forward;
    serverAssert(x &amp;&amp; curscore == x-&gt;score &amp;&amp; sdscmp(x-&gt;ele,ele) == 0);

	//直接更新    
    if ((x-&gt;backward == NULL || x-&gt;backward-&gt;score &lt; newscore) &amp;&amp;
        (x-&gt;level[0].forward == NULL || x-&gt;level[0].forward-&gt;score &gt; newscore))
    {
        x-&gt;score = newscore;
        return x;
    }

    //无法重用旧节点时，删除后重新添加
    zslDeleteNode(zsl, x, update);
    zskiplistNode *newnode = zslInsert(zsl,newscore,x-&gt;ele);
    /* We reused the old node x-&gt;ele SDS string, free the node now
     * since zslInsert created a new one. */
    x-&gt;ele = NULL;
    zslFreeNode(x);
    return newnode;
}
```



# 总结

本文主要对跳表的概念和基本查询思路做了简要分析，对Redis中的跳表的数据结构和基本增删改查接口做了简要的源代码分析。在Redis中的有序集合`zset`就是通过`skiplist`和`dict`组合实现的。另外一处运用则是集群节点中用作内部的数据结构。












---

> Author:   
> URL: http://localhost:1313/posts/%E4%B8%AD%E9%97%B4%E4%BB%B6/redis/redis-zskiplist/  

