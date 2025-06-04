# PHP数组整数键名截断问题


&lt;!--more--&gt;

# 前言

这段时间在做CHY师傅整理的CTF代码审计题目，遇到较难或者比较有意思的题目，都会记下笔记，这次分享一篇关于PHP处理数组时的一个漏洞，这里给出CHY师傅的题目地址：
&gt; [https://github.com/CHYbeta/Code-Audit-Challenges/](https://github.com/CHYbeta/Code-Audit-Challenges/)


# 分析
首先看看PHP官方对这个错误的介绍：
&gt; [https://bugs.php.net/bug.php?id=69892](https://bugs.php.net/bug.php?id=69892)


&gt;var_dump([0 =&gt; 0] === [0x100000000 =&gt; 0]); // bool(true)

下面来看代码：
```PHP
&lt;?php

/*******************************************************************
 * PHP Challenge 2015
 *******************************************************************
 * Why leave all the fun to the XSS crowd?
 *
 * Do you know PHP?
 * And are you up to date with all its latest peculiarities?
 *
 * Are you sure?
 *
 * If you believe you do then solve this challenge and create an
 * input that will make the following code believe you are the ADMIN.
 * Becoming any other user is not good enough, but a first step.
 *
 * Attention this code is installed on a Mac OS X 10.9 system
 * that is running PHP 5.4.30 !!!
 *
 * TIPS: OS X is mentioned because OS X never runs latest PHP
 *       Challenge will not work with latest PHP
 *       Also challenge will only work on 64bit systems
 *       To solve challenge you need to combine what a normal
 *       attacker would do when he sees this code with knowledge
 *       about latest known PHP quirks
 *       And you cannot bruteforce the admin password directly.
 *       To give you an idea - first half is:
 *          orewgfpeowöfgphewoöfeiuwgöpuerhjwfiuvuger
 *
 * If you know the answer please submit it to info@sektioneins.de
 ********************************************************************/

$users = array(
        &#34;0:9b5c3d2b64b8f74e56edec71462bd97a&#34; ,
        &#34;1:4eb5fb1501102508a86971773849d266&#34;,
        &#34;2:facabd94d57fc9f1e655ef9ce891e86e&#34;,
        &#34;3:ce3924f011fe323df3a6a95222b0c909&#34;,
        &#34;4:7f6618422e6a7ca2e939bd83abde402c&#34;,
        &#34;5:06e2b745f3124f7d670f78eabaa94809&#34;,
        &#34;6:8e39a6e40900bb0824a8e150c0d0d59f&#34;,
        &#34;7:d035e1a80bbb377ce1edce42728849f2&#34;,
        &#34;8:0927d64a71a9d0078c274fc5f4f10821&#34;,
        &#34;9:e2e23d64a642ee82c7a270c6c76df142&#34;,
        &#34;10:70298593dd7ada576aff61b6750b9118&#34;
);

$valid_user = false;

$input = $_COOKIE[&#39;user&#39;];
$input[1] = md5($input[1]);

foreach ($users as $user)
{
        $user = explode(&#34;:&#34;, $user);
        if ($input === $user) {
                $uid = $input[0] &#43; 0;
                $valid_user = true;
        }
}

if (!$valid_user) {
        die(&#34;not a valid user\n&#34;);
}

if ($uid == 0) {

        echo &#34;Hello Admin How can I serve you today?\n&#34;;
        echo &#34;SECRETS ....\n&#34;;

} else {
        echo &#34;Welcome back user\n&#34;;
}
?&gt;

```
首先来看看代码逻辑：
&gt; * 从**cookie**中获取**user**
&gt; * 遍历定义的**users**数组，判断传入的数据**input**是否等于它

这里我们要绕过两个条件判断，经过md5解密网站的测试，发现只能解出这一条。
&gt; 06e2b745f3124f7d670f78eabaa94809      //hund

初步可以判断传入的数据为：**Cookie: user[0]=5;user[1]=hund;**
这样传入数据，就可以成功绕过第一个判断，接下来只要让**uid**为0即可。
根据前面给出的数组处理漏洞，分析一下PHP源码**php-src/Zend/zend_hash.c**
```C
//php5.2.14
ZEND_API int zend_hash_compare(HashTable *ht1, HashTable *ht2, compare_func_t compar, zend_bool ordered TSRMLS_DC)
{
	Bucket *p1, *p2 = NULL;
	int result;
	void *pData2;

	IS_CONSISTENT(ht1);
	IS_CONSISTENT(ht2);

	HASH_PROTECT_RECURSION(ht1); 
	HASH_PROTECT_RECURSION(ht2); 

	result = ht1-&gt;nNumOfElements - ht2-&gt;nNumOfElements;
	if (result!=0) {
		HASH_UNPROTECT_RECURSION(ht1); 
		HASH_UNPROTECT_RECURSION(ht2); 
		return result;
	}

	p1 = ht1-&gt;pListHead;
	if (ordered) {
		p2 = ht2-&gt;pListHead;
	}

	while (p1) {
		if (ordered &amp;&amp; !p2) {
			HASH_UNPROTECT_RECURSION(ht1); 
			HASH_UNPROTECT_RECURSION(ht2); 
			return 1; /* That&#39;s not supposed to happen */
		}
		if (ordered) {
			if (p1-&gt;nKeyLength==0 &amp;&amp; p2-&gt;nKeyLength==0) { /* numeric indices */
				result = p1-&gt;h - p2-&gt;h;
				if (result!=0) {
					HASH_UNPROTECT_RECURSION(ht1); 
					HASH_UNPROTECT_RECURSION(ht2); 
					return result;
				}
			} else { /* string indices */
				result = p1-&gt;nKeyLength - p2-&gt;nKeyLength;
				if (result!=0) {
					HASH_UNPROTECT_RECURSION(ht1); 
					HASH_UNPROTECT_RECURSION(ht2); 
					return result;
				}
				result = memcmp(p1-&gt;arKey, p2-&gt;arKey, p1-&gt;nKeyLength);
				if (result!=0) {
					HASH_UNPROTECT_RECURSION(ht1); 
					HASH_UNPROTECT_RECURSION(ht2); 
					return result;
				}
			}
			pData2 = p2-&gt;pData;
		} else {
			if (p1-&gt;nKeyLength==0) { /* numeric index */
				if (zend_hash_index_find(ht2, p1-&gt;h, &amp;pData2)==FAILURE) {
					HASH_UNPROTECT_RECURSION(ht1); 
					HASH_UNPROTECT_RECURSION(ht2); 
					return 1;
				}
			} else { /* string index */
				if (zend_hash_quick_find(ht2, p1-&gt;arKey, p1-&gt;nKeyLength, p1-&gt;h, &amp;pData2)==FAILURE) {
					HASH_UNPROTECT_RECURSION(ht1); 
					HASH_UNPROTECT_RECURSION(ht2); 
					return 1;
				}
			}
		}
		result = compar(p1-&gt;pData, pData2 TSRMLS_CC);
		if (result!=0) {
			HASH_UNPROTECT_RECURSION(ht1); 
			HASH_UNPROTECT_RECURSION(ht2); 
			return result;
		}
		p1 = p1-&gt;pListNext;
		if (ordered) {
			p2 = p2-&gt;pListNext;
		}
	}
	
	HASH_UNPROTECT_RECURSION(ht1); 
	HASH_UNPROTECT_RECURSION(ht2); 
	return 0;
}
```


bucket结构体：
```C
typedef struct bucket {
	ulong h;						/* Used for numeric indexing */
	uint nKeyLength;
	void *pData;
	void *pDataPtr;
	struct bucket *pListNext;
	struct bucket *pListLast;
	struct bucket *pNext;
	struct bucket *pLast;
	char arKey[1]; /* Must be last element */
} Bucket;
```
关键点就在**result = p1-&gt;h - p2-&gt;h**，在**unsigned long**转为**int**时会少掉4个字节。
4294967296转为int就为0。
最终payload：
&gt; **Cookie: user[4294967296]=5;user[1]=hund;**


# 总结

不断的前进

---

> Author:   
> URL: http://localhost:1313/posts/web%E5%AE%89%E5%85%A8/phpbug-69892/  

