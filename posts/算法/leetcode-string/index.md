# Leetcode-字符串刷题记录




### 520. 检测大写字母

```java
public boolean detectCapitalUse(String word) {
        int i = 0;
        char[] arr = word.toCharArray();
        int up = 0, low = 0;
        while(i &lt; arr.length){
            if(arr[i] &gt;= &#39;a&#39;){
                low&#43;&#43;;
            }else{
                up&#43;&#43;;
            }
            i&#43;&#43;;
        }
        if(up == arr.length) return true;
        if(low == arr.length) return true;
        if(up == 1 &amp;&amp; arr[0] &lt; &#39;a&#39;) return true;
        return false;
    }
```



### 125. 验证回文串

经典问题，需要过滤非字符非数字。

```java
public boolean isPalindrome(String s) {
        if(s == null) return true;
        s = s.toLowerCase();
        StringBuilder str = new StringBuilder(s.length());
        for (char c : s.toCharArray()) {
            if ((c &gt;= &#39;0&#39; &amp;&amp; c &lt;= &#39;9&#39;) || (c &gt;= &#39;a&#39; &amp;&amp; c &lt;= &#39;z&#39;)) {
                str.append(c);
            }
        }
        return str.toString().equals(str.reverse().toString());
    }
```







### 14. 最长公共前缀

- 首先计算出最短字符串的长度，这个长度就是外层循环的遍历次数。
- 内层逐个比较字符串的指定位置的值。
- 通过StringBuilder拼接公共前缀。

```java
public String longestCommonPrefix(String[] strs) {
        if(strs.length == 0) 
            return &#34;&#34;;
        StringBuilder sb = new StringBuilder();
        int minLength = 1000;
        //计算最短字符串长度
        for(String str : strs){
            if(str.length() &lt; minLength){
                minLength = str.length();
            }
        }
        String demo = strs[0];
        for(int i = 0; i&lt; minLength; i&#43;&#43;){
            boolean flag = true;
            char c = demo.charAt(i);
            for(int j = 1; j &lt; strs.length; j&#43;&#43;){
                if(strs[j].charAt(i) != c){
                    //不等于
                    flag = false;
                    break;
                }
            }
            if(flag){
                sb.append(c);
                flag = true;
            }else{
                break;
            }
        }
        return sb.toString();
    }
```







### 434. 字符串中的单词数



一轮遍历即可，判断依据是：**当前值是空格，前一位非空格**

```java
public int countSegments(String s) {
        if (s == null) {
            return 0;
        }
        s = s.trim();
        if (s.length() == 0) {
            return 0;
        }
        int count = 1;
        for (int i = 1; i &lt; s.length(); i&#43;&#43;) {
            // 当前值是空格，前一位非空格
            if (s.charAt(i) == 32 &amp;&amp; s.charAt(i - 1) != 32) {
                count&#43;&#43;;
            }
        }
        return count;
    }
```





### 58. 最后一个单词的长度



```java
public int lengthOfLastWord(String s) {
        String[] s1 = s.split(&#34; &#34;);
        if(s1.length == 0) return 0;
        return s1[s1.length -1].length() == 0 ? 0 : s1[s1.length -1].length();
    }
```





### 344. 反转字符串

```java
public void reverseString(char[] s) {
        int left = 0, right = s.length -1;
        while(left &lt; right){
            swap(s , left, right);
            left&#43;&#43;;
            right--;
        }
    }

    public void swap(char[] s, int i, int j){
        char tmp = s[i];
        s[i] = s[j];
        s[j] = tmp;
    }
```





### 541 反转字符串Ⅱ

```java
public String reverseStr(String s, int k) {
        char[] arr = s.toCharArray();
        for(int i = 0; i &lt; arr.length; i&#43;= 2*k){
            //每隔2k反转一次
            //判断剩余字符
            int rest = arr.length - i;
            if(rest &lt; k){
               reverse(arr,i, arr.length - 1); 
            }else if(rest &lt; 2 * k &amp;&amp; rest &gt;= k){
                reverse(arr,i, i &#43; k - 1);
            }else{
                reverse(arr,i, i&#43;k - 1);
            }
        }
        return String.valueOf(arr);
    }


    public void reverse(char[] s,int i, int j) {
        int left = i, right = j;
        while(left &lt; right){
            swap(s , left, right);
            left&#43;&#43;;
            right--;
        }
    }

    public void swap(char[] s, int i, int j){
        char tmp = s[i];
        s[i] = s[j];
        s[j] = tmp;
    }
```





### 557. 反转字符串中的单词Ⅲ



```java
public String reverseWords(String s) {
        String[] strs = s.split(&#34; &#34;);
        String[] reverseString = new String [strs.length];
        int index = 0;
        for(String str : strs){
            reverseString[index&#43;&#43;] = new StringBuffer(str).reverse().toString();
        }
        StringBuilder sb = new StringBuilder();
        for(int i =0; i &lt; strs.length; i&#43;&#43;){
            sb.append(reverseString[i]);
            if(i != reverseString.length - 1) sb.append(&#34; &#34;);
        }
        return sb.toString();
    }
```





### 151. 反转字符串中的单词

```java
public String reverseWords(String s) {
        s = s.trim();
        List&lt;String&gt; wordList = Arrays.asList(s.split(&#34;\\s&#43;&#34;));
        Collections.reverse(wordList);
        return String.join(&#34; &#34;, wordList);
    }
```





### 387. 字符串中的第一个唯一字符

```java
public static int firstUniqChar(String s) {
        int[] letter=new int[26];
        for (char c:s.toCharArray()) {
            letter[c-&#39;a&#39;]&#43;&#43;;
        }
        for (int i = 0; i &lt;s.length() ; i&#43;&#43;) {
            if(letter[s.charAt(i)-&#39;a&#39;]==1) {
                return i;
            }
        }
        return -1;
    }
```



### 389. 找不同

```C&#43;&#43;
char findTheDifference(string s, string t) {
        vector&lt;int&gt; count(26,0);
        for(char ch : s){
            count[ch - &#39;a&#39;]&#43;&#43;;
        }
        for(char ch : t){
            count[ch - &#39;a&#39;]--;
            if(count[ch - &#39;a&#39;] &lt; 0) return ch;
        }
        return &#39; &#39;;
    }
```





### 383. 赎金信

```C&#43;&#43;
bool canConstruct(string ransomNote, string magazine) {
        vector&lt;int&gt; count(26, 0);
        for(char ch : magazine){
            count[ch - &#39;a&#39;]&#43;&#43;;
        }
        for(char ch : ransomNote){
            count[ch - &#39;a&#39;]--;
            if(count[ch - &#39;a&#39;] &lt; 0){
                return false;
            }
        }
        return true;
    }
```





### 242. 有效的字母异位词

```C&#43;&#43;
bool isAnagram(string s, string t) {
        vector&lt;int&gt; cnt(26,0);
        for(char ch : s) cnt[ch - &#39;a&#39;]&#43;&#43;;
        for(char ch : t) cnt[ch - &#39;a&#39;]--;
        for(int count : cnt) if(count != 0) return false;
        return true;
    }
```





### 49. 字母异位词分组

```C&#43;&#43;
vector&lt;vector&lt;string&gt;&gt; groupAnagrams(vector&lt;string&gt;&amp; strs) {
        vector&lt;vector&lt;string&gt;&gt; res;
        map&lt;string, vector&lt;string&gt;&gt; map;
        for(int i =0; i &lt; strs.size(); i&#43;&#43;){
            string key = strs[i];
            sort(key.begin(), key.end());
            map[key].push_back(strs[i]);
        }
        for(auto ite = map.begin(); ite != map.end(); ite&#43;&#43;) res.push_back(ite-&gt;second);
        return res;
    }
```





### 451. 根据字符出现频率排序

```C&#43;&#43;
string frequencySort(string s) {
        map&lt;char,int&gt; map;
        for(char c : s) map[c]&#43;&#43;;
        //sort
        vector&lt;pair&lt;char, int&gt;&gt; vec(map.begin(), map.end());
        sort(vec.begin(), vec.end(), [](auto &amp; a, auto &amp; b){
            return a.second &gt; b.second;
        });
        string res;
        for(auto v : vec) res &#43;= string(v.second,v.first);
        return res;
    }
```





### 657. 机器人能否返回原点

```C&#43;&#43;
bool judgeCircle(string moves) {
        int index = 0;
        int updown = 0, rightleft = 0;
        while(index &lt;= moves.length()){
            switch(moves[index]){
                case &#39;R&#39;:
                    rightleft&#43;&#43;;
                    break;
                case &#39;L&#39;:
                    rightleft--;
                    break;
                case &#39;U&#39;:
                    updown&#43;&#43;;
                    break;
                case &#39;D&#39;:
                    updown--;
                    break;
            }
            index&#43;&#43;;
        }
        if(updown == 0 &amp;&amp; rightleft == 0) return true;
        return false;
    }
```





### 551. 学生出勤记录Ⅰ



```C&#43;&#43;
bool checkRecord(string s) {
        int late = 0, absent = 0;
        for(int i = 0; i &lt; s.length(); i&#43;&#43;){
            if(s[i] == &#39;A&#39;){
                //absent
                absent&#43;&#43;;
                late = 0;
                if(absent &gt; 1) return false;
            }else if(s[i] == &#39;L&#39;){
                late&#43;&#43;;
                if(late &gt; 2) return false;
            }else {
                //P
                late = 0;
            }
        }
        return true;
    }
```



### 28. 实现strStr()

```C&#43;&#43;
int strStr(string haystack, string needle) {
        if ( needle == &#34;&#34; )
            return 0;
        int hlen = haystack.length();
        int nlen = needle.length();
        int i,j;
        for(i=0;i&lt;hlen;i&#43;&#43;){
            for(j=0;j&lt;nlen;j&#43;&#43;)
                if(haystack[i&#43;j]!=needle[j])
                    break;      //不符合就结束本轮匹配
            if(j==nlen)
                return i;
        }
        return -1;
    }
```





### 8. 字符串转换整数

```C&#43;&#43;
int myAtoi(string s) {
        int i=0; // 遍历的游标
        while(i&lt;s.size()&amp;&amp;s[i]==&#39; &#39;) i&#43;&#43;; // 去掉空白字符
        if(i==s.size()) return 0; // 如果s为全空
        
        // 处理第一个非空字符
        int sign=1;
        if(s[i]==&#39;-&#39;){
            sign=-1;
            i&#43;&#43;;
        }
        else if(s[i]==&#39;&#43;&#39;)
            i&#43;&#43;;
        else if(!isdigit(s[i]))
            return 0;

        // 处理第一个数字字符
        int n=0;
        while(isdigit(s[i])&amp;&amp;i&lt;s.size()){
            if((INT_MAX-(s[i]-&#39;0&#39;))/10.0&lt;n) return sign==-1?sign*INT_MAX-1:INT_MAX; // 如果溢出
            n=10*n&#43;(s[i]-&#39;0&#39;);
            i&#43;&#43;;
        }
        return sign*n;
    }
```



### 67. 二进制求和



```C&#43;&#43;
string addBinary(string a, string b) {
        string result;
        const int n = a.size() &gt; b.size() ? a.size() : b.size();
        reverse(a.begin(), a.end());
        reverse(b.begin(), b.end());
        int carry = 0;
        for(int i = 0; i &lt; n; i&#43;&#43;){
            int ai = i &lt; a.size() ? a[i] - &#39;0&#39; : 0;
            int bi = i &lt; b.size() ? b[i] - &#39;0&#39; : 0;
            int val = (ai &#43; bi &#43; carry) % 2;
            carry = (ai &#43; bi &#43; carry) / 2;
            result.insert(result.begin(), val &#43; &#39;0&#39;);
        }
        if(carry == 1) result.insert(result.begin(), &#39;1&#39;);
        return result;
    }
```



### 412. Fizz Buzz



```C&#43;&#43;
vector&lt;string&gt; fizzBuzz(int n) {
        vector&lt;string&gt; res;
        for(int i = 1; i &lt;= n; i&#43;&#43;){
            string tmp;
            if(i % 15 == 0){
                res.push_back(&#34;FizzBuzz&#34;);
            }else if(i % 5 == 0){
                res.push_back(&#34;Buzz&#34;);
            }else if(i % 3 == 0){
                res.push_back(&#34;Fizz&#34;);
            }else{
                res.push_back(to_string(i));
            }
            
        }
        return res;
    }
```





### 299. 猜数字游戏

需要注意数字出现的次数。

```C&#43;&#43;
string getHint(string secret, string guess) {
        int bull = 0, cow = 0;
        vector&lt;int&gt; num(10,0);
        for(int i = 0; i &lt; guess.size(); i&#43;&#43;){
            if(guess[i] == secret[i]){
                bull&#43;&#43;;
            }else {
                //当前位置不一致，需要判断当前字符是否出现在secret里面
                //通过vector存储数字出现次数
                if(num[secret[i] - &#39;0&#39;]&#43;&#43; &lt; 0) cow&#43;&#43;;
                if(num[guess[i] - &#39;0&#39;]-- &gt; 0) cow&#43;&#43;;
            }
        }
        return to_string(bull) &#43; &#39;A&#39; &#43; to_string(cow) &#43; &#39;B&#39;;
    }
```



### 506. 相对名次

```C&#43;&#43;
vector&lt;string&gt; findRelativeRanks(vector&lt;int&gt;&amp; score) {
        vector&lt;int&gt; org = score;
        sort(score.rbegin(), score.rend());
        unordered_map&lt;int, string&gt; order;
        for(int i = 0; i &lt; score.size(); i&#43;&#43;){
            if(i &gt;= 3) order[score[i]] = to_string(i&#43; 1);
            if(i == 0) order[score[i]] = &#34;Gold Medal&#34;;
            if (i == 1) order[score[i]] = &#34;Silver Medal&#34;;
            if (i == 2) order[score[i]] = &#34;Bronze Medal&#34;;
        }
        vector&lt;string&gt; res(score.size(), &#34;&#34;);
        for(int i = 0; i &lt; res.size(); i&#43;&#43;){
            res[i] = order[org[i]];
        }
        return res;
    }
```



### 539. 最小时间差

- 将时间转换成分钟。
- 注意钟表是模运算。

```C&#43;&#43;
int findMinDifference(vector&lt;string&gt;&amp; timePoints) {
        vector&lt;int&gt; q;
        int res = INT_MAX;
        for(string t : timePoints){
            int hour = 10 * (t[0] - &#39;0&#39;) &#43; t[1] - &#39;0&#39;;
            int min = 10 * (t[3] - &#39;0&#39;) &#43; t[4] - &#39;0&#39;;
            q.push_back(60 * hour &#43; min);
        }
        sort(q.begin(), q.end());
        q.push_back(q[0] &#43; 24 * 60);
        for(int i = 1; i &lt; q.size(); i&#43;&#43;){
            res = min(res, q[i] - q[i - 1]);
        }
        return res;
    }
```





### 553. 最优除法

题解：**被除数不变，除数最小**，如果能理解这个数学知识，那么就是一道easy难度的题目。

```C&#43;&#43;
string optimalDivision(vector&lt;int&gt;&amp; nums) {
        if(nums.size() == 1){
            return to_string(nums[0]);
        }else if(nums.size() == 2){
            return to_string(nums[0]) &#43; &#34;/&#34; &#43; to_string(nums[1]);
        }
        string res(to_string(nums[0]) &#43; &#34;/(&#34;);
        for(int i = 1; i &lt; nums.size() - 1; i&#43;&#43;){
            res &#43;= to_string(nums[i]) &#43; &#34;/&#34;;
        }
        res &#43;= to_string(nums.back()) &#43; &#34;)&#34;;
        return res;
    }
```





### 38. 外观数列

```C&#43;&#43;
public String countAndSay(int n) {
        String res = &#34;1&#34;;
        for(int i = 2; i &lt;= n; i&#43;&#43;){
            StringBuilder temp = new StringBuilder();
            for(int j = 0; j &lt; res.length(); j&#43;&#43;){
                int count = 1;
                while(j &#43; 1 &lt; res.length() &amp;&amp; res.charAt(j) == res.charAt(j &#43; 1)){
                    j&#43;&#43;;
                    count&#43;&#43;;
                }
                temp.append(count).append(res.charAt(j));
            }
            res = temp.toString();
        }
        return res;
    }
```



---

> Author:   
> URL: http://localhost:1313/posts/%E7%AE%97%E6%B3%95/leetcode-string/  

