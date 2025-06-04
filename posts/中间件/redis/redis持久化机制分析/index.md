# Redis持久化机制分析




## 简介

Redis数据保存在内存中，为了防止进程异常退出而导致数据丢失，可以考虑将数据保存到磁盘上。



Redis提供的两种持久化方案：

- RDB：直接将数据库状态写到文件，既可以手动执行，也可以配置为定期执行。
- AOF：将写命令append到AOF文件中。



## RDB



两种方案：

- save：直接执行
- bgsave：fork一个子进程执行



`rdbSave`是save的具体实现函数，bgsave与其区别是通过fork子进程完成备份，并在完成后给父进程发送信号。

```c
int rdbSave(char *filename) {
    dictIterator *di = NULL;
    dictEntry *de;
    char tmpfile[256];
    char magic[10];
    int j;
    long long now = mstime();
    FILE *fp;
    rio rdb;
    uint64_t cksum;

    // 创建临时文件
    snprintf(tmpfile,256,&#34;temp-%d.rdb&#34;, (int) getpid());
    fp = fopen(tmpfile,&#34;w&#34;);
    if (!fp) {
        redisLog(REDIS_WARNING, &#34;Failed opening .rdb for saving: %s&#34;,
            strerror(errno));
        return REDIS_ERR;
    }

    // 初始化 I/O
    rioInitWithFile(&amp;rdb,fp);

    // 设置校验和函数
    if (server.rdb_checksum)
        rdb.update_cksum = rioGenericUpdateChecksum;

    // 写入 RDB 版本号
    snprintf(magic,sizeof(magic),&#34;REDIS%04d&#34;,REDIS_RDB_VERSION);
    if (rdbWriteRaw(&amp;rdb,magic,9) == -1) goto werr;

    // 遍历所有数据库
    for (j = 0; j &lt; server.dbnum; j&#43;&#43;) {

        // 指向数据库
        redisDb *db = server.db&#43;j;

        // 指向数据库键空间
        dict *d = db-&gt;dict;

        // 跳过空数据库
        if (dictSize(d) == 0) continue;

        // 创建键空间迭代器
        di = dictGetSafeIterator(d);
        if (!di) {
            fclose(fp);
            return REDIS_ERR;
        }

        /* Write the SELECT DB opcode 
         *
         * 写入 DB 选择器
         */
        if (rdbSaveType(&amp;rdb,REDIS_RDB_OPCODE_SELECTDB) == -1) goto werr;
        if (rdbSaveLen(&amp;rdb,j) == -1) goto werr;

        /* Iterate this DB writing every entry 
         *
         * 遍历数据库，并写入每个键值对的数据
         */
        while((de = dictNext(di)) != NULL) {
            sds keystr = dictGetKey(de);
            robj key, *o = dictGetVal(de);
            long long expire;
            
            // 根据 keystr ，在栈中创建一个 key 对象
            initStaticStringObject(key,keystr);

            // 获取键的过期时间
            expire = getExpire(db,&amp;key);

            // 保存键值对数据
            if (rdbSaveKeyValuePair(&amp;rdb,&amp;key,o,expire,now) == -1) goto werr;
        }
        dictReleaseIterator(di);
    }
    di = NULL; /* So that we don&#39;t release it again on error. */

    /* EOF opcode 
     *
     * 写入 EOF 代码
     */
    if (rdbSaveType(&amp;rdb,REDIS_RDB_OPCODE_EOF) == -1) goto werr;

    /* CRC64 checksum. It will be zero if checksum computation is disabled, the
     * loading code skips the check in this case. 
     *
     * CRC64 校验和。
     *
     * 如果校验和功能已关闭，那么 rdb.cksum 将为 0 ，
     * 在这种情况下， RDB 载入时会跳过校验和检查。
     */
    cksum = rdb.cksum;
    memrev64ifbe(&amp;cksum);
    rioWrite(&amp;rdb,&amp;cksum,8);

    /* Make sure data will not remain on the OS&#39;s output buffers */
    // 冲洗缓存，确保数据已写入磁盘
    if (fflush(fp) == EOF) goto werr;
    if (fsync(fileno(fp)) == -1) goto werr;
    if (fclose(fp) == EOF) goto werr;

    /* Use RENAME to make sure the DB file is changed atomically only
     * if the generate DB file is ok. 
     *
     * 使用 RENAME ，原子性地对临时文件进行改名，覆盖原来的 RDB 文件。
     */
    if (rename(tmpfile,filename) == -1) {
        redisLog(REDIS_WARNING,&#34;Error moving temp DB file on the final destination: %s&#34;, strerror(errno));
        unlink(tmpfile);
        return REDIS_ERR;
    }

    // 写入完成，打印日志
    redisLog(REDIS_NOTICE,&#34;DB saved on disk&#34;);

    // 清零数据库脏状态
    server.dirty = 0;

    // 记录最后一次完成 SAVE 的时间
    server.lastsave = time(NULL);

    // 记录最后一次执行 SAVE 的状态
    server.lastbgsave_status = REDIS_OK;

    return REDIS_OK;

werr:
    // 关闭文件
    fclose(fp);
    // 删除文件
    unlink(tmpfile);

    redisLog(REDIS_WARNING,&#34;Write error saving DB on disk: %s&#34;, strerror(errno));

    if (di) dictReleaseIterator(di);

    return REDIS_ERR;
}
```



### RDB载入还原

Redis对RDB文件的载入还原步骤如下：

- 判断是否开启AOF持久化功能，若开启载入AOF文件
- 否则载入RDB文件



之所以优先载入AOF文件，是因为：**AOF更新频率比RDB更高**



### 定期执行

Redis允许通过设置服务器配置项使server每隔一段时间自动执行一次BGSAVE命令。

```
save &lt;seconds&gt; &lt;changes&gt; [&lt;seconds&gt; &lt;changes&gt; ...]
save 900 1		//900s之内，进行了至少1次修改操作
save 300 5		//300s之内，进行了至少10次修改操作
save 60 2		//60s之内，进行了至少2次修改操作
```

以上的配置项保存在`struct redisServer&gt;saveparams`字段中，这是一个数组，`saveparam`的结构如下：

```c
// 服务器的保存条件（BGSAVE 自动执行的条件）
struct saveparam {

    // 多少秒之内
    time_t seconds;

    // 发生多少次修改
    int changes;

};
```



同时server状态还维持了两个相关的变量：

- dirty：修改计数器，记录距离上次执行save指令，服务器对数据库状态进行了多少次修改。
- lastsave：UNIX时间戳，记录上次成功执行SAVE的时间。



Redis中的定时任务函数`serverCron`其中由于一项工作就是遍历`saveparams`选项的条件是否满足，满足的话则执行。



### 文件结构

按照顺序结构如下：

- REDIS：5 bytes，为REDIS五个字符，快速检查载入的文件是否为RDB文件
- db_version：4 bytes，字符串表示的整数，记录RDB文件的版本号
- databases：包含多个数据库以及对应数据
  - 每个非空数据库都可以保存SELECTDB、db_number、key_value_pairs。
- EOF：1 byte，标志着正文内容的结束
- check_sum：8 bytes的无符号整数，是对前四个部分计算得出的，载入文件时会对比check_sum，以此检查文件是否出错。



```c
root@pc:/opt/redis# od -c dump.rdb 
0000000   R   E   D   I   S   0   0   1   0 372  \t   r   e   d   i   s
0000020   -   v   e   r  \v   2   5   5   .   2   5   5   .   2   5   5
0000040 372  \n   r   e   d   i   s   -   b   i   t   s 300   @ 372 005
0000060   c   t   i   m   e 302 314 213 354   b 372  \b   u   s   e   d
0000100   -   m   e   m 302   @ 032 020  \0 372  \b   a   o   f   -   b
0000120   a   s   e 300  \0 376  \0 373 001  \0  \0 005   h   e   l   l
0000140   o 005   v   a   l   u   e 377   &#43; 361   .   &#39; 337 354  \n   0
0000160
```



## AOF

AOF是Redis的另一种持久化方案，这是通过保存写命令来记录数据库的状态，恢复时从前到后执行一次即可恢复数据。所有的命令都是以Redis的命令请求协议的格式来保存的。




### 配置项
```
# 是否开启AOF，默认关闭（no）
appendonly yes

# 指定 AOF 文件名
appendfilename appendonly.aof

# Redis支持三种不同的刷盘模式：
# appendfsync always #每次发生数据变化立即写入到磁盘中
# appendfsync no  #sync由OS控制
appendfsync everysec #每秒钟写入磁盘一次

#设置为yes表示rewrite期间对新写操作不fsync,暂时存在内存中,等rewrite完成后再写入，默认为no
no-appendfsync-on-rewrite no 

#当前AOF文件大小是上次日志重写得到AOF文件大小的二倍时，自动启动新的日志重写过程。
auto-aof-rewrite-percentage 100

#当前AOF文件启动新的日志重写过程的最小值，避免刚刚启动Reids时由于文件尺寸较小导致频繁的重写。
auto-aof-rewrite-min-size 64mb
```



### AOF实现

AOF的实现可以分为以下三个步骤：

- append：每次写命令执行完成后，会以协议格式将命令追加到`struct redisServer-&gt;aof_buf`中
- write和sync：`serverCron`中会判断`appendfsync`的值来选择`flushAppendOnlyFile`的具体策略。
  - always：将`aof_buf`中的所有内容write并sync到AOF文件
  - everysec：将`aof_buf`中的所有内容write到AOF文件，如果上次sync时间距离现在超过一秒钟，会fork一个进程再次对AOF文件进行sync。
  - no：将`aof_buf`中的所有内容同步写入到AOF文件，sync动作由OS决定。



```c
	// 根据 AOF 政策，
    // 考虑是否需要将 AOF 缓冲区中的内容写入到 AOF 文件中
    /* AOF postponed flush: Try at every cron cycle if the slow fsync
     * completed. */
    if (server.aof_flush_postponed_start) flushAppendOnlyFile(0);

    /* AOF write errors: in this case we have a buffer to flush as well and
     * clear the AOF error in case of success to make the DB writable again,
     * however to try every second is enough in case of &#39;hz&#39; is set to
     * an higher frequency. */
    run_with_period(1000) {
        if (server.aof_last_write_status == REDIS_ERR)
            flushAppendOnlyFile(0);
    }
```



### AOF载入还原

Redis对AOF文件的载入还原步骤如下：

- 创建一个不带网络连接的fake client。
- 从AOF文件中分析并读取一条写命令，使用fake client执行，直至所有写命令全部执行完成。



### AOF重写

为了解决AOF文件大小膨胀的问题，Redis提供了AOF重写功能：Redis服务器可以创建一个新的AOF文件来替代现有的AOF文件。新文件不会包含冗余命令，类似于Bitcask模型的merge操作，会对冗余指令进行合并，重写由**子进程**执行，并不会阻塞主线程。



为了防止重写期间，服务器继续处理写命令请求而导致数据不一致的问题，设置了一个重写缓冲区，server端接收到命令后，会将写命令同时发送给AOF缓冲区和重写缓冲区。

子进程完成AOF重写后，会给父进程发送一个信号，父进程的信号处理函数执行如下逻辑：

1. 将重写缓冲区中所有内容写入新的AOF文件中，保证数据一致性。
2. 对新的AOF文件改名，原子地覆盖原文件。



## RDB和AOF的对比


- 启动优先级：RDB低，AOF高
- 体积：RDB小，AOF大
- 恢复速度：RDB快
- 数据安全性：RDB丢数据，AOF根据策略决定
- 轻重：RDB较重


Redis4.0提出了**混合使用**AOF和RDB的方法，RDB以一定频率执行，两次RDB执行期间，使用AOF记录两次之间的所有命令操作，第二次只需要将AOF中修改的数据进行更新即可。


---

> Author:   
> URL: http://localhost:1313/posts/%E4%B8%AD%E9%97%B4%E4%BB%B6/redis/redis%E6%8C%81%E4%B9%85%E5%8C%96%E6%9C%BA%E5%88%B6%E5%88%86%E6%9E%90/  

