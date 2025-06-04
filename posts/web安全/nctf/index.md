# 南京邮电大学CTF平台Web系列WriteUp

&lt;!--more--&gt;

## 签到1

直接查看源代码，得到flag
nctf{flag_admiaanaaaaaaaaaaa}

## 签到2
要求输入zhimakaimen
审查元素更改输入框type为text 长度改为11
提交得到nctf{follow_me_to_exploit}

## 这题不是Web
这题真的不是Web
下载图片，以文本方式打开，最后一行即为flag

## 层层递进
burpsuite抓包之后，扫出一个404.html
源代码如下
&lt;!-- more --&gt;
``` html

&lt;!DOCTYPE HTML PUBLIC &#34;-//W3C//DTD HTML 4.01//EN&#34; &#34;http://www.w3.org/TR/html4/strict.dtd&#34;&gt;
&lt;HTML&gt;&lt;HEAD&gt;&lt;TITLE&gt;有人偷偷先做题，哈哈飞了吧？&lt;/TITLE&gt;
&lt;META HTTP-EQUIV=&#34;Content-Type&#34; Content=&#34;text/html; charset=GB2312&#34;&gt;
&lt;STYLE type=&#34;text/css&#34;&gt;
  BODY { font: 9pt/12pt 宋体 }
  H1 { font: 12pt/15pt 宋体 }
  H2 { font: 9pt/12pt 宋体 }
  A:link { color: red }
  A:visited { color: maroon }
&lt;/STYLE&gt;
&lt;/HEAD&gt;&lt;BODY&gt;
&lt;center&gt;
&lt;TABLE width=500 border=0 cellspacing=10&gt;&lt;TR&gt;&lt;TD&gt;
&lt;!-- Placed at the end of the document so the pages load faster --&gt;
&lt;!--  
&lt;script src=&#34;./js/jquery-n.7.2.min.js&#34;&gt;&lt;/script&gt;
&lt;script src=&#34;./js/jquery-c.7.2.min.js&#34;&gt;&lt;/script&gt;
&lt;script src=&#34;./js/jquery-t.7.2.min.js&#34;&gt;&lt;/script&gt;
&lt;script src=&#34;./js/jquery-f.7.2.min.js&#34;&gt;&lt;/script&gt;
&lt;script src=&#34;./js/jquery-{.7.2.min.js&#34;&gt;&lt;/script&gt;
&lt;script src=&#34;./js/jquery-t.7.2.min.js&#34;&gt;&lt;/script&gt;
&lt;script src=&#34;./js/jquery-h.7.2.min.js&#34;&gt;&lt;/script&gt;
&lt;script src=&#34;./js/jquery-i.7.2.min.js&#34;&gt;&lt;/script&gt;
&lt;script src=&#34;./js/jquery-s.7.2.min.js&#34;&gt;&lt;/script&gt;
&lt;script src=&#34;./js/jquery-_.7.2.min.js&#34;&gt;&lt;/script&gt;
&lt;script src=&#34;./js/jquery-i.7.2.min.js&#34;&gt;&lt;/script&gt;
&lt;script src=&#34;./js/jquery-s.7.2.min.js&#34;&gt;&lt;/script&gt;
&lt;script src=&#34;./js/jquery-_.7.2.min.js&#34;&gt;&lt;/script&gt;
&lt;script src=&#34;./js/jquery-a.7.2.min.js&#34;&gt;&lt;/script&gt;
&lt;script src=&#34;./js/jquery-_.7.2.min.js&#34;&gt;&lt;/script&gt;
&lt;script src=&#34;./js/jquery-f.7.2.min.js&#34;&gt;&lt;/script&gt;
&lt;script src=&#34;./js/jquery-l.7.2.min.js&#34;&gt;&lt;/script&gt;
&lt;script src=&#34;./js/jquery-4.7.2.min.js&#34;&gt;&lt;/script&gt;
&lt;script src=&#34;./js/jquery-g.7.2.min.js&#34;&gt;&lt;/script&gt;
&lt;script src=&#34;./js/jquery-}.7.2.min.js&#34;&gt;&lt;/script&gt;
--&gt;

&lt;p&gt;来来来，听我讲个故事：&lt;/p&gt;
&lt;ul&gt;
&lt;li&gt;从前，我是一个好女孩，我喜欢上了一个男孩小A。&lt;/li&gt;
&lt;li&gt;有一天，我终于决定要和他表白了！话到嘴边，鼓起勇气...
&lt;/li&gt;
&lt;li&gt;可是我却又害怕的&lt;a href=&#34;javascript:history.back(1)&#34;&gt;后退&lt;/a&gt;了。。。&lt;/li&gt;
&lt;/ul&gt;
&lt;h2&gt;为什么？&lt;br&gt;为什么我这么懦弱？&lt;/h2&gt;
&lt;hr&gt;
&lt;p&gt;最后，他居然向我表白了，好开森...说只要骗足够多的笨蛋来这里听这个蠢故事浪费时间，&lt;/p&gt;
&lt;p&gt;他就同意和我交往！&lt;/p&gt;
&lt;p&gt;谢谢你给出的一份支持！哇哈哈\(^o^)/~！&lt;/p&gt;

&lt;/TD&gt;&lt;/TR&gt;&lt;/TABLE&gt;
&lt;/center&gt;
&lt;/BODY&gt;&lt;/HTML&gt;
```
注意引入的js文件名

## 单身二十年
burpsuite爬行出一个search_key.php，flag藏在Response里面

## MYSQL
打开首先提示robots.txt，打开发现以下代码

``` php
TIP:sql.php

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
半吊子PHP水平分析一波，intval()函数是将变量转化为整数，需要得到的id变量为1024，但不允许输入的值为1024，因为有intval()函数，可以输入1024.1a，从而得到flag。

## COOKIE
TIP: 0==not
给出的提示以上信息，题目又是cookie，查看cookie发现它的值为0，将它改成1，得到flag:nctf{cookie_is_different_from_session}
![nctf](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/nctf1.png)



---

> Author:   
> URL: http://localhost:1313/posts/web%E5%AE%89%E5%85%A8/nctf/  

