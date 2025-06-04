# 理解PHP变量覆盖漏洞


&lt;!--more--&gt;

## 0x01 简介

变量覆盖漏洞是指攻击者使用自定义的变量去覆盖源代码中的变量，从而改变代码逻辑，实现攻击目的的一种漏洞。

通常造成变量覆盖的往往是以下几种情况：
- register_globals=On （PHP5.3废弃，PHP5.4中已经正式移除此功能）
- 可变变量
- extract()
- parse_str()
- import_request_variables()
下面正式介绍变量覆盖的具体利用方法与原理。

## 0x02 正文

### $可变变量
看下面一段代码：
```PHP
&lt;?php
foreach (array(&#39;_COOKIE&#39;,&#39;_POST&#39;,&#39;_GET&#39;) as $_request)  
{
    foreach ($$_request as $_key=&gt;$_value)  
    {
        $$_key=  $_value;
    }
}
$id = isset($id) ? $id : 2;
if($id == 1) {
    echo &#34;flag{xx}&#34;;
    die();
}
?&gt;
```
使用foreach遍历COOKIE，POST，GET数组，然后将数组键名作为变量名，键值作为变量值。上述代码传入id=1即可得到flag。


### extract()
首先看看手册中对该函数的定义
![](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/php-cover/cover1.jpg)

下面从实例讲起:
```PHP
&lt;?php
$flag=&#39;flag{xxxxx}&#39;; 
extract($_GET);
 if(isset($a)) { 
    $content=trim(file_get_contents($flag));
    if($a==$content)
    { 
        echo $flag; 
    }
   else
   { 
    echo&#39;Oh.no&#39;;
   } 
   }else {
    echo &#34;input a value&#34;;
   }
?&gt;
```
首先extract函数将所有GET方法得到的数组键名与键值转化为内部变量与值，接可以看到content变量使用file_get_contents()读入一个文件，由于trim()函数处理后，content变量变为一个空字符串，代码逻辑只需要**a == content**即可。

传入 **?a=** 即可得到flag。

### parse_str()
![](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/cover3.jpg)

```PHP
&lt;?php
parse_str(&#34;a=1&#34;);
echo $a.&#34;&lt;br/&gt;&#34;;      //$a=1
parse_str(&#34;b=1&amp;c=2&#34;,$myArray);
print_r($myArray);   //Array ( [b] =&gt; 1 [c] =&gt; 2 ) 
?&gt;
```
这个简单的实例中可以看出parse_str可以将字符串解析为变量与值。

### import_request_variables()

![](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/import.jpg)

```PHP
&lt;?php
// 此处将导入 GET 和 POST 变量
// 使用&#34;rvar_&#34;作为前缀
$rvar_foo = 1;
import_request_variables(&#34;gP&#34;, &#34;rvar_&#34;);
echo $rvar_foo;
?&gt; 
```
通过import_request_variables函数作用后，可以对原变量值进行覆盖。

## 0x03 总结
博客文章以后会在公众号上同步发布，欢迎师傅们关注。微信号：coding_lin

---

> Author:   
> URL: http://localhost:1313/posts/web%E5%AE%89%E5%85%A8/php-audit-cover/  

