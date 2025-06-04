# Kafka源码分析(五) - 索引细节



# Intro


LogSegment日志对象管理以下三种Index：
1. 位移索引文件(.index)
2. 时间戳索引文件(.timeindex)
3. 已中止事物的索引文件(.txnindex)

```java
private final LazyIndex&lt;OffsetIndex&gt; lazyOffsetIndex;  
private final LazyIndex&lt;TimeIndex&gt; lazyTimeIndex;  
private final TransactionIndex txnIndex;
```


# AbstractIndex


AbstractIndex作为索引中的顶层抽象接口，封装索引结构中一些通用的属性和处理逻辑：

```java
protected final ReentrantLock lock = new ReentrantLock();  
  
//当前文件的开始offset  
private final long baseOffset;  
  
//最大索引大小  
private final int maxIndexSize;  
  
//是否可写  
private final boolean writable;  
  
//映射索引文件  
private volatile File file;  
  
// 索引文件长度  
private volatile long length;  
  
//mmap 内存映射buffer  
private volatile MappedByteBuffer mmap;  
  
//最大索引entry数量  
private volatile int maxEntries;  
  
//索引entry数量  
private volatile int entries;
```


抽象方法：

```java
/**  
 * Do a basic sanity check on this index to detect obvious problems * 
 * @throws CorruptIndexException if any problems are found  
 */
public abstract void sanityCheck();  
  
/**  
 * Remove all entries from the index which have an offset greater than or equal to the given offset. 
 * Truncating to an offset larger than the largest in the index has no effect. */
public abstract void truncateTo(long offset);  
  
/**  
 * Remove all the entries from the index. 
 * */
protected abstract void truncate();

//不同entry大小
protected abstract int entrySize();


/**  
 * To parse an entry in the index. * 
 * @param buffer the buffer of this memory mapped index.  
 * @param n the slot  
 * @return the index entry stored in the given slot.  
 */
 protected abstract IndexEntry parseEntry(ByteBuffer buffer, int n);
```


## 初始化逻辑

```java
public AbstractIndex(File file, long baseOffset, int maxIndexSize, boolean writable) throws IOException {  
    Objects.requireNonNull(file);  
    this.file = file;  
    this.baseOffset = baseOffset;  
    this.maxIndexSize = maxIndexSize;  
    this.writable = writable;  
  
    createAndAssignMmap();  
    this.maxEntries = mmap.limit() / entrySize();  
    this.entries = mmap.position() / entrySize();  
}
```

createAndAssignMmap方法会初始化file和mmap对象，实际加载index文件使用mmap方式，将内核buffer映射到用户buffer，减少了内存拷贝的开销，这也是Kafka较为重要的优化点之一：

```java
private void createAndAssignMmap() throws IOException {  
  
    //1.1 创建文件并构建RandomAccessFile  
    boolean newlyCreated = file.createNewFile();  
    RandomAccessFile raf;  
    if (writable)  
        raf = new RandomAccessFile(file, &#34;rw&#34;);  
    else  
        raf = new RandomAccessFile(file, &#34;r&#34;);  
  
    try {  
        /* pre-allocate the file if necessary */  
        //1.2 若文件不存在，检查大小并设置文件长度  
        if (newlyCreated) {  
            if (maxIndexSize &lt; entrySize())  
                throw new IllegalArgumentException(&#34;Invalid max index size: &#34; &#43; maxIndexSize);  
            raf.setLength(roundDownToExactMultiple(maxIndexSize, entrySize()));  
        }  
  
        //1.3 调用createMappedBuffer构建MappedByteBuffer  
        long length = raf.length();  
        MappedByteBuffer mmap = createMappedBuffer(raf, newlyCreated, length, writable, entrySize());  
  
        this.length = length;  
        this.mmap = mmap;  
    } finally {  
        Utils.closeQuietly(raf, &#34;index &#34; &#43; file.getName());  
    }  
}
```


初始化mmap：

```java
private static MappedByteBuffer createMappedBuffer(RandomAccessFile raf, boolean newlyCreated, long length,  
                                                   boolean writable, int entrySize) throws IOException {  
    MappedByteBuffer idx;  
    //1.1 通过FileChannel获取mappedByteBuffer  
    if (writable)  
        idx = raf.getChannel().map(FileChannel.MapMode.READ_WRITE, 0, length);  
    else  
        idx = raf.getChannel().map(FileChannel.MapMode.READ_ONLY, 0, length);  
  
    /* set the position in the index for the next entry */  
    //1.2 配置position  
    if (newlyCreated)  
        idx.position(0);  
    else  
        // if this is a pre-existing index, assume it is valid and set position to last entry  
        idx.position(roundDownToExactMultiple(idx.limit(), entrySize));  
  
    return idx;  
}
```


## 查找


在查找Index中使用的二分查找算法， 但是Kafka对此做了特定优化：由于读取的索引数据通常都是文件尾部的，因此通过将数据划分为冷热区域，查找时将范围确定在热区内，减少了二分查找的工作量。

```java
private int indexSlotRangeFor(ByteBuffer idx, long target, IndexSearchType searchEntity,  
                              SearchResultType searchResultType) {  
    // check if the index is empty  
    if (entries == 0)  
        return -1;  
  
  
    //1.1 获取当前第一个热区entry offset  
    int firstHotEntry = Math.max(0, entries - 1 - warmEntries());  
    // 1.2 检查目标offset，是否在热区范围内  
    if (compareIndexEntry(parseEntry(idx, firstHotEntry), target, searchEntity) &lt; 0) {  
        //1.3 缩减二分查找范围在热区范围内  
        return binarySearch(idx, target, searchEntity,  
            searchResultType, firstHotEntry, entries - 1);  
    }  
  
    //1.4 全量查找前检查offset是否小于最小offset  
    // check if the target offset is smaller than the least offset    
    if (compareIndexEntry(parseEntry(idx, 0), target, searchEntity) &gt; 0) {  
        switch (searchResultType) {  
            case LARGEST_LOWER_BOUND:  
                return -1;  
            case SMALLEST_UPPER_BOUND:  
                return 0;  
        }  
    }  
  
    return binarySearch(idx, target, searchEntity, searchResultType, 0, firstHotEntry);  
}
```

冷热区域划分规则：

```java
protected final int warmEntries() {  
    return 8192 / entrySize();  
}
```

至于选取8192的原因，在注释中解释得很清楚：
1. 4096几乎是所有CPU架构的page cache大小，如果再小无法保证覆盖更多的热数据。
2. 8KB索引信息大约对应4MB的日志信息（offset index）或2.7MB（time index），这已经满足热区需求。


二分查找算法：

```java
private int binarySearch(ByteBuffer idx, long target, IndexSearchType searchEntity,  
                         SearchResultType searchResultType, int begin, int end) {  
    // binary search for the entry  
    int lo = begin;  
    int hi = end;  
    while (lo &lt; hi) {  
        int mid = (lo &#43; hi &#43; 1) &gt;&gt;&gt; 1;  
        IndexEntry found = parseEntry(idx, mid);  
        int compareResult = compareIndexEntry(found, target, searchEntity);  
        if (compareResult &gt; 0)  
            hi = mid - 1;  
        else if (compareResult &lt; 0)  
            lo = mid;  
        else  
            return mid;  
    }  
    switch (searchResultType) {  
        case LARGEST_LOWER_BOUND:  
            return lo;  
        case SMALLEST_UPPER_BOUND:  
            if (lo == entries - 1)  
                return -1;  
            else  
                return lo &#43; 1;  
        default:  
            throw new IllegalStateException(&#34;Unexpected searchResultType &#34; &#43; searchResultType);  
    }  
}
```

# LazyIndex

LazyIndex作为AbstractIndex的包装类，主要起到lazy-load作用，初始化时并不会读取index文件完成加载，而是在第一次读取索引时进行加载：

```java
public T get() throws IOException {  
    IndexWrapper wrapper = indexWrapper;  
    if (wrapper instanceof IndexValue&lt;?&gt;)  
        return ((IndexValue&lt;T&gt;) wrapper).index;  
    else {  
        lock.lock();  
        try {  
            if (indexWrapper instanceof IndexValue&lt;?&gt;)  
                return ((IndexValue&lt;T&gt;) indexWrapper).index;  
            else if (indexWrapper instanceof IndexFile) {  
                IndexFile indexFile = (IndexFile) indexWrapper;  
                IndexValue&lt;T&gt; indexValue = new IndexValue&lt;&gt;(loadIndex(indexFile.file));  
                indexWrapper = indexValue;  
                return indexValue.index;  
            } else  
                throw new IllegalStateException(&#34;Unexpected type for indexWrapper &#34; &#43; indexWrapper.getClass());  
        } finally {  
            lock.unlock();  
        }  
    }  
}
```


```java
private T loadIndex(File file) throws IOException {  
    switch (indexType) {  
        case OFFSET:  
            return (T) new OffsetIndex(file, baseOffset, maxIndexSize, true);  
        case TIME:  
            return (T) new TimeIndex(file, baseOffset, maxIndexSize, true);  
        default:  
            throw new IllegalStateException(&#34;Unexpected indexType &#34; &#43; indexType);  
    }  
}
```


# 索引项


索引项提供的接口IndexEntry如下，包含两个字段，分别key、value：

```java
public interface IndexEntry {  
    long indexKey();  
    long indexValue();  
}
```


当前有offset index和time index两种实现：


```java
public final class OffsetPosition implements IndexEntry {  
    public final long offset;  
    public final int position;
}
```


```java
public class TimestampOffset implements IndexEntry {  
  
    public static final TimestampOffset UNKNOWN = new TimestampOffset(-1, -1);  
  
    public final long timestamp;  
    public final long offset;
}
```



# OffsetIndex &amp; TimeIndex

## append

两者写入逻辑较为类似，通过mmap方式分别追加写入key和value。索引项的写入时机通过`index.interval.bytes`配置项控制，每当写入多少数据量时写入一次索引项，这部分逻辑可以在`org.apache.kafka.storage.internals.log.LogSegment#append`中找到。

offset index写入索引项流程：

```java
public void append(long offset, int position) {  
    lock.lock();  
    try {  
        //1.1 检查索引文件是否写满  
        if (isFull())  
            throw new IllegalArgumentException(&#34;Attempt to append to a full index (size = &#34; &#43; entries() &#43; &#34;).&#34;);  
  
        //1.2 如果当前文件为空，或者新添加的offset大于当前最后一个offset，则添加索引  
        if (entries() == 0 || offset &gt; lastOffset) {  
            log.trace(&#34;Adding index entry {} =&gt; {} to {}&#34;, offset, position, file().getAbsolutePath());  
              
            //1.3 写入相对offset和position  
            mmap().putInt(relativeOffset(offset));  
            mmap().putInt(position);  
              
            //1.4 更新entries数量和lastOffset  
            incrementEntries();  
            lastOffset = offset;  
            if (entries() * ENTRY_SIZE != mmap().position())  
                throw new IllegalStateException(entries() &#43; &#34; entries but file position in index is &#34; &#43; mmap().position());  
        } else  
            throw new InvalidOffsetException(&#34;Attempt to append an offset &#34; &#43; offset &#43; &#34; to position &#34; &#43; entries() &#43;  
                &#34; no larger than the last offset appended (&#34; &#43; lastOffset &#43; &#34;) to &#34; &#43; file().getAbsolutePath());  
    } finally {  
        lock.unlock();  
    }  
}
```


time index写入索引项流程：

```java
public void maybeAppend(long timestamp, long offset, boolean skipFullCheck) {  
    lock.lock();  
    try {  
        if (!skipFullCheck &amp;&amp; isFull())  
            throw new IllegalArgumentException(&#34;Attempt to append to a full time index (size = &#34; &#43; entries() &#43; &#34;).&#34;);  
        
        if (entries() != 0 &amp;&amp; offset &lt; lastEntry.offset)  
            throw new InvalidOffsetException(&#34;Attempt to append an offset (&#34; &#43; offset &#43; &#34;) to slot &#34; &#43; entries()  
                &#43; &#34; no larger than the last offset appended (&#34; &#43; lastEntry.offset &#43; &#34;) to &#34; &#43; file().getAbsolutePath());  
        if (entries() != 0 &amp;&amp; timestamp &lt; lastEntry.timestamp)  
            throw new IllegalStateException(&#34;Attempt to append a timestamp (&#34; &#43; timestamp &#43; &#34;) to slot &#34; &#43; entries()  
                &#43; &#34; no larger than the last timestamp appended (&#34; &#43; lastEntry.timestamp &#43; &#34;) to &#34; &#43; file().getAbsolutePath());  
  
        // We only append to the time index when the timestamp is greater than the last inserted timestamp.  
        // If all the messages are in message format v0, the timestamp will always be NoTimestamp. In that case, the time        // index will be empty.        
        if (timestamp &gt; lastEntry.timestamp) {  
            log.trace(&#34;Adding index entry {} =&gt; {} to {}.&#34;, timestamp, offset, file().getAbsolutePath());  
            MappedByteBuffer mmap = mmap();  
            mmap.putLong(timestamp);  
            mmap.putInt(relativeOffset(offset));  
            incrementEntries();  
            this.lastEntry = new TimestampOffset(timestamp, offset);  
            if (entries() * ENTRY_SIZE != mmap.position())  
                throw new IllegalStateException(entries() &#43; &#34; entries but file position in index is &#34; &#43; mmap.position());  
        }  
    } finally {  
        lock.unlock();  
    }  
}
```


# Reference

[https://strimzi.io/blog/2021/12/17/kafka-segment-retention/](https://strimzi.io/blog/2021/12/17/kafka-segment-retention/)
[https://learn.conduktor.io/kafka/kafka-topics-internals-segments-and-indexes/](https://learn.conduktor.io/kafka/kafka-topics-internals-segments-and-indexes/)


---

> Author:   
> URL: http://localhost:1313/posts/%E4%B8%AD%E9%97%B4%E4%BB%B6/kafka/kafka-source-code-5-index-details/  

