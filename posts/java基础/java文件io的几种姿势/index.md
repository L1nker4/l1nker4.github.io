# Java File IO的几种姿势




![Java File IO 复杂度](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/image-20230426224504720.png)



## 读写方式



Java中的File IO基本可以分为三类：

- 面向字节传输的传统IO方式
- FileChannel文件通道
- mmap内存映射



### Buffered Stream I/O

BufferedReader通过内部buffer将文件的一部分字节缓存下来，减少频繁的磁盘IO。

```java
public static void testStream(){
        try (BufferedReader reader = Files.newBufferedReader(Paths.get(PATH), StandardCharsets.UTF_8)) {
            String line = null;
            while ((line = reader.readLine()) != null) {
                System.out.println(line);
            }
        } catch (IOException x) {
            System.err.format(&#34;IOException: %s%n&#34;, x);
        }
    }
```





### Unbuffered Streams I/O

```java
public static void testUnStream() {
        try (InputStream in = Files.newInputStream(Paths.get(PATH));
             BufferedReader reader =
                     new BufferedReader(new InputStreamReader(in))) {
            String line = null;
            while ((line = reader.readLine()) != null) {
                System.out.println(line);
            }
        } catch (IOException x) {
            System.err.println(x);
        }
    }
```



### Channel I/O

```java
public static byte[] readAllBytes(Path path) throws IOException {
        try (SeekableByteChannel sbc = Files.newByteChannel(path);
             InputStream in = Channels.newInputStream(sbc)) {
            long size = sbc.size();
            if (size &gt; (long)MAX_BUFFER_SIZE)
                throw new OutOfMemoryError(&#34;Required array size too large&#34;);

            return read(in, (int)size);
        }
    }


public static SeekableByteChannel newByteChannel(Path path, OpenOption... options)
        throws IOException
    {
        Set&lt;OpenOption&gt; set = new HashSet&lt;OpenOption&gt;(options.length);
        Collections.addAll(set, options);
        return newByteChannel(path, set);
    }


public static SeekableByteChannel newByteChannel(Path path,
                                                     Set&lt;? extends OpenOption&gt; options,
                                                     FileAttribute&lt;?&gt;... attrs)
        throws IOException
    {
        return provider(path).newByteChannel(path, options, attrs);
    }

public abstract SeekableByteChannel newByteChannel(Path path,
        Set&lt;? extends OpenOption&gt; options, FileAttribute&lt;?&gt;... attrs) throws IOException;
```





```java
public static void testChannel() {
        try (SeekableByteChannel sbc = Files.newByteChannel(Paths.get(PATH))) {
            final int BUFFER_CAPACITY = 10;
            ByteBuffer buf = ByteBuffer.allocate(BUFFER_CAPACITY);

            // Read the bytes with the proper encoding for this platform. If
            // you skip this step, you might see foreign or illegible
            // characters.
            String encoding = System.getProperty(&#34;file.encoding&#34;);
            while (sbc.read(buf) &gt; 0) {
                buf.flip();
                System.out.print(Charset.forName(encoding).decode(buf));
                buf.clear();
            }
        }catch (IOException e){
            System.err.println(e);
        }
    }
```



### FileChannel

```java
public static void testFileChannel() {
        try {
            FileChannel fileChannel = new RandomAccessFile(new File(PATH), &#34;rw&#34;).getChannel();
            ByteBuffer byteBuffer = ByteBuffer.allocate(4);
            fileChannel.read(byteBuffer);
            String str = new String(byteBuffer.array(), StandardCharsets.UTF_8);
            System.out.println(str);
        }catch (IOException e){
            System.err.println(e);
        }
    }
```





### mmap

适用于读写小文件的应用场景。

```java
public static void testMmap() {
        try {
            FileChannel fileChannel = new RandomAccessFile(new File(PATH), &#34;rw&#34;).getChannel();
            MappedByteBuffer mappedByteBuffer = fileChannel.map(FileChannel.MapMode.READ_WRITE, 0, fileChannel.size());
            byte[] data = new byte[4];
            mappedByteBuffer.get(data);
            String str = new String(data, StandardCharsets.UTF_8);
            System.out.println(str);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```



使用mmap方式对的几个缺点：

1. 适用于文件大小确定的使用场景，因为使用前必须指定好内存映射的大小，一次map的大小限制在1.5G左右，重复map存在虚拟内存的多次分配回收。
2. mmap内存回收的方式繁琐。





## Reference

[https://docs.oracle.com/javase/tutorial/essential/io/file.html](https://docs.oracle.com/javase/tutorial/essential/io/file.html)

[https://docs.oracle.com/javase/tutorial/essential/io/rafs.html](https://docs.oracle.com/javase/tutorial/essential/io/rafs.html)


---

> Author:   
> URL: http://localhost:1313/posts/java%E5%9F%BA%E7%A1%80/java%E6%96%87%E4%BB%B6io%E7%9A%84%E5%87%A0%E7%A7%8D%E5%A7%BF%E5%8A%BF/  

