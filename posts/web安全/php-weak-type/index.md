# PHP弱类型产生的安全问题


&lt;!--more--&gt;

# 简介

PHP作为一种弱类型编程语言，在定义变量时无需像C&#43;&#43;等强类型时定义数据类型。
```PHP
&lt;?php
$a = 1;
$a = &#34;hello&#34;;
$a = [];
?&gt;
```
上述代码可以正常运行，在PHP中可以随时将变量改成其他数据类型。下面再来看一个例子：
```PHP
&lt;?php
$a = &#39;1&#39;;   //a现在是字符串&#39;1&#39;
$a *= 2;    //a现在是整数2
?&gt;
```
下面就从具体的实例中总结一下弱类型产生的安全问题。
# 比较运算符

## 简介
PHP有这些比较运算符
```PHP
&lt;?php
$a == $b;   //相等
$a === $b;  //全等
$a != $b;   //不等
$a &lt;&gt; $b;   //不等
$a !== $b;   //不全等
$a &lt; $b;   //小于
$a &gt; $b;   //大于
$a &lt;= $b;   //小于等于
$a &gt;= $b;   //大于等于
$a &lt;==&gt; $b;   //太空船运算符：当$a小于、等于、大于$b时分别返回一个小于、等于、大于0的integer 值。 PHP7开始提供.  
$a ?? $b ?? $c;   //NULL 合并操作符 从左往右第一个存在且不为 NULL 的操作数。如果都没有定义且不为 NULL，则返回 NULL。PHP7开始提供。 
?&gt;
```
存在安全问题的一般都是运算符在类型转换时产生的。给出PHP手册中比较的表格。
&gt; http://php.net/manual/zh/types.comparisons.php

下面看一段从PHP手册中摘的一段：
&gt;如果该字符串没有包含 &#39;.&#39;，&#39;e&#39; 或 &#39;E&#39; 并且其数字值在整型的范围之内（由 PHP_INT_MAX 所定义），该字符串将被当成 integer 来取值。其它所有情况下都被作为 float 来取值。 

```PHP
&lt;?php
$foo = 1 &#43; &#34;10.5&#34;;                // $foo is float (11.5)
$foo = 1 &#43; &#34;-1.3e3&#34;;              // $foo is float (-1299)
$foo = 1 &#43; &#34;bob-1.3e3&#34;;           // $foo is integer (1)
$foo = 1 &#43; &#34;bob3&#34;;                // $foo is integer (1)
$foo = 1 &#43; &#34;10 Small Pigs&#34;;       // $foo is integer (11)
$foo = 4 &#43; &#34;10.2 Little Piggies&#34;; // $foo is float (14.2)
$foo = &#34;10.0 pigs &#34; &#43; 1;          // $foo is float (11)
$foo = &#34;10.0 pigs &#34; &#43; 1.0;        // $foo is float (11)     
?&gt; 
```

可以看到字符串中如果有&#39;.&#39;或者e(E)，字符串将会被解析为整数或者浮点数。这些特性在处理Hash字符串时会产生一些安全问题。

```PHP
&lt;?php
&#34;0e132456789&#34;==&#34;0e7124511451155&#34; //true
&#34;0e123456abc&#34;==&#34;0e1dddada&#34;	//false
&#34;0e1abc&#34;==&#34;0&#34;     //true
&#34;0admin&#34; == &#34;0&#34;     //ture

&#34;0x1e240&#34;==&#34;123456&#34;		//true
&#34;0x1e240&#34;==123456		//true
&#34;0x1e240&#34;==&#34;1e240&#34;		//false
?&gt;
```
若Hash字符串以0e开头，在进行!=或者==运算时将会被解析为科学记数法，即0的次方。
或字符串为0x开头，将会被当作十六进制的数。

下面给出几个以0e开头的MD5加密之后的密文。
```
QNKCDZO
0e830400451993494058024219903391
  
s1885207154a
0e509367213418206700842008763514
  
s1836677006a
0e481036490867661113260034900752
  
s155964671a
0e342768416822451524974117254469
  
s1184209335a
0e072485820392773389523109082030
```
下面从具体函数理解弱类型带来的问题。

# 具体函数

## md5()
```PHP
&lt;?php
if (isset($_GET[&#39;Username&#39;]) &amp;&amp; isset($_GET[&#39;password&#39;])) {
    $logined = true;
    $Username = $_GET[&#39;Username&#39;];
    $password = $_GET[&#39;password&#39;];

     if (!ctype_alpha($Username)) {$logined = false;}
     if (!is_numeric($password) ) {$logined = false;}
     if (md5($Username) != md5($password)) {$logined = false;}
     if ($logined){
    echo &#34;successful&#34;;
      }else{
           echo &#34;login failed!&#34;;
        }
    }
?&gt;
```
要求输入的username为字母，password为数字，并且两个变量的MD5必须一样，这时就可以考虑以0e开头的MD5密文。**md5(&#39;240610708&#39;) == md5(&#39;QNKCDZO&#39;)** 可以成功得到flag。

## sha1()
```PHP
&lt;?php

$flag = &#34;flag&#34;;

if (isset($_GET[&#39;name&#39;]) and isset($_GET[&#39;password&#39;])) 
{
    if ($_GET[&#39;name&#39;] == $_GET[&#39;password&#39;])
        echo &#39;&lt;p&gt;Your password can not be your name!&lt;/p&gt;&#39;;
    else if (sha1($_GET[&#39;name&#39;]) === sha1($_GET[&#39;password&#39;]))
      die(&#39;Flag: &#39;.$flag);
    else
        echo &#39;&lt;p&gt;Invalid password.&lt;/p&gt;&#39;;
}
else
    echo &#39;&lt;p&gt;Login first!&lt;/p&gt;&#39;;
?&gt;
```

sha1函数需要传入的数据类型为字符串类型，如果传入数组类型则会返回NULL

```php
var_dump(sha1([1]));	//NULL
```



因此传入**name[]=1&amp;password[]=2** 即可成功绕过

## strcmp()   php version &lt; 5.3
```PHP
&lt;?php
    $password=&#34;***************&#34;
     if(isset($_POST[&#39;password&#39;])){

        if (strcmp($_POST[&#39;password&#39;], $password) == 0) {
            echo &#34;Right!!!login success&#34;;n
            exit();
        } else {
            echo &#34;Wrong password..&#34;;
        }
?&gt;
```
和sha1()函数一样，strcmp()是字符串处理函数，如果给strcmp()函数传入数组，无论数据是否相等都会返回0，当然这只存在与PHP版本小于5.3之中。

## intval()
```PHP
&lt;?php
if($_GET[id]) {
   mysql_connect(SAE_MYSQL_HOST_M . &#39;:&#39; . SAE_MYSQL_PORT,SAE_MYSQL_USER,SAE_MYSQL_PASS);
  mysql_select_db(SAE_MYSQL_DB);
  $id = intval($_GET[id]);
  $query = @mysql_fetch_array(mysql_query(&#34;select content from ctf2 where id=&#39;$id&#39;&#34;));
  if ($_GET[id]==1024) {
      echo &#34;&lt;p&gt;no! try again&lt;/p&gt;&#34;;
  }
  else{
    echo($query[content]);
  }
}
?&gt;
```
intval()函数获取变量的整数值,程序要求从浏览器获得的id值不为1024，因此输入**1024.1**即可获得flag。

## json_decode()

```PHP
&lt;?php
if (isset($_POST[&#39;message&#39;])) {
    $message = json_decode($_POST[&#39;message&#39;]);
    $key =&#34;*********&#34;;
    if ($message-&gt;key == $key) {
        echo &#34;flag&#34;;
    } 
    else {
        echo &#34;fail&#34;;
    }
 }
 else{
     echo &#34;~~~~&#34;;
 }
?&gt;
```
需要POST的message中的key值等于程序中的key值，如果传入的key为整数0，那么在比较运算符==的运算下，将会判定字符串为整数0，那么只需要传入数据：**message={&#34;key&#34;:0}**

## array_search() 和  in_array()
上面两个函数和前面d额是一样的问题，会存在下面的一些转换为题：
&gt; &#34;admin&#34; == 0; //true
&gt; &#34;1admin&#34; == 1; //true

# 总结
熬夜写文章已经很晚了，就懒得再写总结了。以后每周总结一篇关于代码审计的小知识点。

---

> Author:   
> URL: http://localhost:1313/posts/web%E5%AE%89%E5%85%A8/php-weak-type/  

