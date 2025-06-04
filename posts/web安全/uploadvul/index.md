# 浅析文件上传漏洞


&lt;!--more--&gt;

# 0x01简介 #

文件上传漏洞简而言之就是攻击者可以通过某些手段上传非法文件到服务器端的一种漏洞，攻击者往往会以此得到webshell，从而进一步提权。为此开发者也会想出各种方法去防止上传漏洞的产生。以下列出几种常见的校验方式：
- 前端JS校验
- content-type字段校验
- 服务器端后缀名校验
- 文件头校验
- 服务器端扩展名校验
  

  下面从几个实例来详细解释上传漏洞的原理与突破方法。

# 0x02正文 #

### 一、前端JS校验 ###
JS校验往往只是通过脚本获得文件的后缀名，再通过白名单验证，以下列出JS代码供参考：
```JavaScript
&lt;script type=&#34;text/javascript&#34;&gt;
    function checkFile() {
        var file = document.getElementsByName(&#39;upload_file&#39;)[0].value;
        if (file == null || file == &#34;&#34;) {
            alert(&#34;请选择要上传的文件!&#34;);
            return false;
        }
        //定义允许上传的文件类型
        var allow_ext = &#34;.jpg|.png|.gif&#34;;
        //提取上传文件的类型
        var ext_name = file.substring(file.lastIndexOf(&#34;.&#34;));
        //判断上传文件类型是否允许上传
        if (allow_ext.indexOf(ext_name) == -1) {
            var errMsg = &#34;该文件不允许上传，请上传&#34; &#43; allow_ext &#43; &#34;类型的文件,当前文件类型为：&#34; &#43; ext_name;
            alert(errMsg);
            return false;
        }
    }
&lt;/script&gt;
```
此种校验方式往往很容易突破，一种是通过抓取HTTP数据包，修改重放便可以突破。或者通过浏览器修改前端代码，从而改变代码逻辑。

### 二、content-type字段校验 ###
首先对MIME进行简单的知识普及
&gt;http://www.cnblogs.com/jsean/articles/1610265.html

&gt;https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Basics_of_HTTP/MIME_Types

```php
&lt;?php
        if($_FILES[&#39;userfile&#39;][&#39;type&#39;] != &#34;image/gif&#34;)  #这里对上传的文件类型进行判断，如果不是image/gif类型便返回错误。
                {   
                 echo &#34;Sorry, we only allow uploading GIF images&#34;;
                 exit;
                 }
         $uploaddir = &#39;uploads/&#39;;
         $uploadfile = $uploaddir . basename($_FILES[&#39;userfile&#39;][&#39;name&#39;]);
         if (move_uploaded_file($_FILES[&#39;userfile&#39;][&#39;tmp_name&#39;], $uploadfile))
             {
                 echo &#34;File is valid, and was successfully uploaded.\n&#34;;
                } else {
                     echo &#34;File uploading failed.\n&#34;;
    }
     ?&gt;
```
此类校验方式绕过也十分简单，burpsuite截取数据包，修改content-type值，发送数据包即可。

### 三、文件头校验 ###

文件头基本概念：
&gt;https://www.cnblogs.com/mq0036/p/3912355.html

上传webshell的过程中,添加文件头便可以突破此类校验。例如
```php
GIF89a&lt;?php phpinfo(); ?&gt;
```

### 四、服务器端扩展名校验 ###
此类校验的上传绕过往往较为复杂，常常与其他漏洞搭配使用，如解析漏洞，文件包含漏洞。

```php
&lt;?php
$type = array(&#34;php&#34;,&#34;php3&#34;);
//判断上传文件类型
$fileext = fileext($_FILE[&#39;file&#39;][&#39;name&#39;]);
if(!in_array($fileext,$type)){
    echo &#34;upload success!&#34;;
}
else{
    echo &#34;sorry&#34;;
}
?&gt;
```
Apache解析漏洞，IIS6.0解析漏洞在此不再过多阐述。

### 五、编辑器上传漏洞 ###
早期编辑器例如FCK,eWeb编辑器正在逐渐淡出市场，但是并没有使富文本编辑器的占用率变低，因此在渗透测试的过程中，一般可以查找编辑器漏洞从而获得webshell。

# 0x03总结 #
在渗透测试的过程中,遇到上传点，通常情况下截取数据包，再通过各种姿势进行bypass。关于防护的几点建议：

- 在服务器端进行校验
- 上传的文件可以进行时间戳md5进行命名
- 上传的路径隐藏
-  文件存储目录权限控制

当然最重要的是，维持网站中间件的安全，杜绝旧版解析漏洞的存在。



---

> Author:   
> URL: http://localhost:1313/posts/web%E5%AE%89%E5%85%A8/uploadvul/  

