# Leetcode-数组刷题记录






## 数组的遍历



### 485. 最大连续1的个数

```java
public int findMaxConsecutiveOnes(int[] nums) {
        //res存储最长的子数组
        int res = 0;
        //count记录的是每个被0隔断的子数组
        int count = 0;
        for(int i = 0; i &lt; nums.length; i&#43;&#43;){
            //如果当前元素是1，更新当前子数组的长度
            if(nums[i] == 1){
                count&#43;&#43;;
            }else {
                //遇到0，比较res, count的大小，并将count重置为0
                res = Math.max(res, count);
                count = 0;
            }
        }
        //避免最后一个子数组是最大的情况，例如：1,0,1,1,0,1,1,1,1
        return Math.max(res, count);
    }
```





### 495. 提莫攻击

```java
public int findPoisonedDuration(int[] timeSeries, int duration) {
        int res = 0;
        for(int i = 0; i &lt; timeSeries.length; i &#43;&#43;){
            //当前能持续的时间
            int time = timeSeries[i] &#43; duration -1;
            //超过下一次攻击的时间
            if(i != timeSeries.length -1 &amp;&amp; time &gt;= timeSeries[i &#43; 1]){
                res &#43;= (timeSeries[i &#43; 1] - timeSeries[i]);
            }else{
                res &#43;= duration;
            }
        }
        return res;
    }
```



### 第三大的数

时间复杂度要求$O(n)$，也就是只能通过一趟遍历完成。

```java
public int thirdMax(int[] nums) {
        if(nums.length == 1){
            return nums[0];
        }
        if(nums.length == 2){
            return nums[0] &gt;= nums[1] ? nums[0] : nums[1];
        }
        long firstMax = Long.MIN_VALUE;
        long secondMax = Long.MIN_VALUE;
        long thirdMax = Long.MIN_VALUE;
    	//判断相等情况是避免相同数字,例如：[2, 2, 3, 1]
        for(int num : nums){
            if(num &gt; firstMax) {
                thirdMax = secondMax;
                secondMax = firstMax;
                firstMax = num;
            }else if(num == firstMax) {
                continue;
            }else if(num &gt; secondMax){
                thirdMax = secondMax;
                secondMax = num;
            }else if(num == secondMax) {
                continue;
            }else if(num &gt; thirdMax){
                thirdMax = num;
            }
        }
        if(thirdMax == Long.MIN_VALUE){
            return (int)firstMax;
        }else{
            return (int) thirdMax;
        }
    }
```





### 628. 三个数的最大乘积



偷🐔解法，需要注意考虑负数的情况。

时间复杂度为$O(nlogN)$。

```java
public int maximumProduct(int[] nums) {
        Arrays.sort(nums);
        int len = nums.length;
                return Math.max(nums[0] * nums[1] * nums[nums.length - 1], 
                                nums[nums.length - 1] * nums[nums.length - 2] * nums[nums.length - 3]);
    }
```



如果使用一趟遍历的方法，那么需要使用五个变量，分别保存最大的三个数，和最小的两个数。





## 统计数组中的元素



### 645. 错误的集合

```java
public int[] findErrorNums(int[] nums) {
        int[] res = new int[2];
        int[] count = new int[nums.length];
        for(int num : nums){
            count[num - 1]&#43;&#43;;
        }
        for(int i = 0; i&lt; count.length;i&#43;&#43;){
            if(count[i] == 2){
                res[0] = i &#43; 1;
            }else if(count[i] == 0){
                res[1] = i &#43; 1;
            }
        }
        return res;
    }
```





### 697. 数组的度

三个**HashMap**的作用分别如下：

- left：数字第一次出现的索引
- right：数字最后一次出现的索引
- count：数字出现的次数

```java
public int findShortestSubArray(int[] nums) {
        Map&lt;Integer, Integer&gt; left = new HashMap(),
            right = new HashMap(), count = new HashMap();

        for (int i = 0; i &lt; nums.length; i&#43;&#43;) {
            int x = nums[i];
            if (left.get(x) == null) left.put(x, i);
            right.put(x, i);
            count.put(x, count.getOrDefault(x, 0) &#43; 1);
        }

        int ans = nums.length;
        int degree = Collections.max(count.values());
        for (int x: count.keySet()) {
            if (count.get(x) == degree) {
                ans = Math.min(ans, right.get(x) - left.get(x) &#43; 1);
            }
        }
        return ans;
    }
```



### 448. 找到所有数组中消失的数字



**HashMap**做记录，再通过一轮遍历找到不存在的数字。

时间复杂度$O(n)$，空间复杂度$O(n)$。

```java
public List&lt;Integer&gt; findDisappearedNumbers(int[] nums) {
        HashMap&lt;Integer, Boolean&gt; map = new HashMap&lt;Integer, Boolean&gt;();
        for (int i = 0; i &lt; nums.length; i&#43;&#43;) {
            map.put(nums[i], true);
        }
        List&lt;Integer&gt; res = new LinkedList&lt;Integer&gt;();
        for (int i = 1; i &lt;= nums.length; i&#43;&#43;) {
            if (!map.containsKey(i)) {
                res.add(i);
            }
        }  
        return res;
    }
```



如果使用时间复杂度$O(n)$，并不用额外的空间，可以通过添加标记来解决。

一轮遍历添加标记：能标识该数字出现过。

第二轮遍历查找没有做标记的数字。

```java
public List&lt;Integer&gt; findDisappearedNumbers(int[] nums) {
        for (int i = 0; i &lt; nums.length; i&#43;&#43;) {
            int index = (nums[i] - 1) % nums.length;
            nums[index] &#43;= nums.length;
        }
        List&lt;Integer&gt; res = new ArrayList&lt;&gt;();
        for(int i = 0; i &lt; nums.length; i&#43;&#43;){
            if(nums[i] &lt;= nums.length){
                res.add(i &#43; 1);
            }
        }
        return res;
    }
```





### 442. 数组中重复的数据



与上面一题异曲同工。使用标记法。

```java
public List&lt;Integer&gt; findDuplicates(int[] nums) {
        for (int i = 0; i &lt; nums.length; i&#43;&#43;) {
            int index = (nums[i] - 1) % nums.length;
            nums[index] &#43;= nums.length;
        }
        List&lt;Integer&gt; res = new ArrayList&lt;&gt;();
        for(int i = 0; i &lt; nums.length; i&#43;&#43;){
            if(nums[i] &gt; nums.length * 2){
                res.add(i &#43; 1);
            }
        }
        return res;
    }
```



###  41. 缺失的第一个正数

通过索引交换元素。

```java
public int firstMissingPositive(int[] nums) {
        for (int i = 0; i &lt; nums.length; i&#43;&#43;) {
            while((nums[i] &gt;= 1 &amp;&amp; nums[i] &lt;= nums.length) &amp;&amp; (nums[i] != nums[nums[i] - 1])){
                int temp = nums[i];
                nums[i] = nums[temp -1];
                nums[temp -1] = temp;
            }
        }
        int res = nums.length &#43; 1;
        for(int i = 0; i &lt; nums.length; i&#43;&#43;){
            if (nums[i] != (i &#43; 1)) {
                res = i &#43; 1;
                break;
            }
        }
        return res;
    }
```







### 274. H指数

偷🐔了一下，直接sort解决。

```java
public int hIndex(int[] citations) {
        int len = citations.length;
        Arrays.sort(citations);
        for(int i = 0; i &lt; len; i&#43;&#43;){
            int count = len - i;
            if(citations[i] &gt;= count){
                return count;
            }
        }
        return 0;
    }
```



## 数组的改变、移动



### 453. 最小操作次数使数组元素相等

每次操作会使$n-1$个元素增加1  == &gt;  每次操作让一个元素减去1

因此需要计算的次数就是：**数组中所有元素（除去最小值）与最小值之间的差值。**

（Python的API用的舒服）

```python
def minMoves(self, nums):
        sum = 0
        minnum = min(nums)
        for i in nums:
            sum &#43;= i - minnum
        return sum
```





### 665. 非递减数列

拐点不能出现两次，修复会有两种情况：

- nums[i - 2] &gt; nums[i]：将nums[i]改为nums[i - 1]
- nums[i - 2] &lt;= nums[i]：将nums[i - 1]改为nums[i]

```java
public boolean checkPossibility(int[] nums) {
        int count = 0;
        for(int i = 1 ; i&lt; nums.length; i&#43;&#43;){
            if(nums[i - 1] &gt; nums[i]){
                count&#43;&#43;;
                if(count &gt;= 2) return false;
                if(i - 2 &gt;= 0 &amp;&amp; nums[i - 2] &gt; nums[i]){
                    nums[i] = nums[i - 1];
                }else{
                    nums[i - 1] = nums[i];
                }
            }
        }
        return true;
}
```





### 283. 移动零

通过`index`存储有效位。

```java
public void moveZeroes(int[] nums) {
        if (nums == null || nums.length == 0) {
            return;
        }
        int index = 0;
        for (int i = 0; i &lt; nums.length; i&#43;&#43;) {
            if (nums[i] != 0) {
                nums[index&#43;&#43;] = nums[i];
            }
        }
        while (index &lt; nums.length) {
            nums[index&#43;&#43;] = 0;
        }
    }
```





## 二维数组及滚动数组



### 118. 杨辉三角



```java
public List&lt;List&lt;Integer&gt;&gt; generate(int numRows) {
       List&lt;List&lt;Integer&gt;&gt; ans = new ArrayList&lt;List&lt;Integer&gt;&gt;();
        for(int i = 0; i &lt; numRows; i&#43;&#43;){
            List&lt;Integer&gt; list = new ArrayList&lt;&gt;();
            list.add(1);
            for(int j = 1; j &lt; i; j&#43;&#43;){
                List&lt;Integer&gt; front = ans.get(i - 1);
                list.add(front.get(j) &#43; front.get(j -1));
            }
            if(i &gt; 0){
                list.add(1);
            }
            ans.add(list);
        }
        return ans;
    }
```



### 119. 杨辉三角Ⅱ

```java
public List&lt;Integer&gt; getRow(int rowIndex) {
        List&lt;List&lt;Integer&gt;&gt; ans = new ArrayList&lt;List&lt;Integer&gt;&gt;();
        for(int i = 0; i &lt;= rowIndex; i&#43;&#43;){
            List&lt;Integer&gt; list = new ArrayList&lt;&gt;();
            list.add(1);
            for(int j = 1; j &lt; i; j&#43;&#43;){
                List&lt;Integer&gt; front = ans.get(i - 1);
                list.add(front.get(j) &#43; front.get(j -1));
            }
            if(i &gt; 0){
                list.add(1);
            }
            ans.add(list);
        }
        return ans.get(rowIndex);
    }
```





### 598. 范围求和Ⅱ

```java
public int maxCount(int m, int n, int[][] ops) {
        int minX = m , minY = n;
        for(int i = 0; i &lt; ops.length; i&#43;&#43;){
            minX = Math.min(minX, ops[i][0]);
            minY = Math.min(minY, ops[i][1]);
        }
        return minX * minY;
    }
```



### 419. 甲板上的战舰

暴力判断：如果当前点是X，判断其左边和上边是否有X，没有则不连通，增加计数。

```C
public int countBattleships(char[][] board) {
        int count = 0;
        for(int i = 0; i &lt;board.length; i&#43;&#43;){
            for(int j = 0; j &lt; board[i].length; j&#43;&#43;){
                if(board[i][j] == &#39;X&#39;){
                    if(i &gt; 0 &amp;&amp; board[i - 1][j] == &#39;X&#39; || j &gt; 0 &amp;&amp; board[i][j - 1] == &#39;X&#39;) continue;
                    count&#43;&#43;;
                }
            }
        }
        return count;
    }
```





### 189. 数组的旋转

三轮reverse。

```java
public void rotate(int[] nums, int k) {
        //如果数组长度小于旋转次数，那么执行k次和执行k % length次结果一样
        k %= nums.length;
        reverse(nums, 0, nums.length - 1);
        reverse(nums, 0, k - 1);
        reverse(nums, k, nums.length - 1);
    }

    private void reverse(int[] nums, int start, int end) {
        while (start &lt; end) {
            int tmp = nums[start];
            nums[start] = nums[end];
            nums[end] = tmp;
            start&#43;&#43;;
            end--;
        }
    }
```



### 396. 旋转函数

可通过数学归纳法得到$F[k]= F[k-1] &#43; sum - n*A[n-k]$

```java
public int maxRotateFunction(int[] A) {
        int n = A.length;
        int max = 0;
        int count = 0;
        // 统计数组所有数的和
        int sum = 0;
        // 计算 F(1) 的值
        for (int i : A) {
            max &#43;= count&#43;&#43; * i;
            sum &#43;= i;
        }
        int tmp = max;
        for (int i = 1; i &lt; n; i&#43;&#43;) {
            tmp = tmp &#43; sum - n * A[n - i];
            max = Math.max(tmp, max);
        }
        return max;
    }
```





## 特定顺序遍历二维数组



### 54. 螺旋矩阵

抓住两个index的变化趋势，按照规律变化就行。

```java
public List&lt;Integer&gt; spiralOrder(int[][] matrix) {
        List&lt;Integer&gt; res = new ArrayList&lt;&gt;();
        int m = matrix.length, n = matrix[0].length;
        int up = 0, down = m - 1, left = 0, right = n -1;
        while(true){
            for(int i = left; i &lt;= right; i&#43;&#43;) res.add(matrix[up][i]);
            if(&#43;&#43;up &gt; down) break;
            for(int i = up; i &lt;= down; i&#43;&#43;) res.add(matrix[i][right]);
            if (--right &lt; left) break;
            for(int i = right; i &gt;= left; i--) res.add(matrix[down][i]);
            if (--down &lt; up) break;
            for(int i = down; i &gt;= up; i--) res.add(matrix[i][left]);
            if(&#43;&#43;left &gt; right) break;
        }
        return res;
    }
```





### 59. 螺旋矩阵Ⅱ

和上题一样。

```java
public int[][] generateMatrix(int n) {
        int[][] matrix = new int[n][n];
        int up = 0, down = n - 1, left = 0, right = n -1;
        int num = 1;
        while(true){
            for(int i = left; i &lt;= right; i&#43;&#43;) matrix[up][i] = num&#43;&#43;;
            if(&#43;&#43;up &gt; down) break;
            for(int i = up; i &lt;= down; i&#43;&#43;) matrix[i][right] = num&#43;&#43;;
            if (--right &lt; left) break;
            for(int i = right; i &gt;= left; i--) matrix[down][i] = num&#43;&#43;;
            if (--down &lt; up) break;
            for(int i = down; i &gt;= up; i--) matrix[i][left] = num&#43;&#43;;
            if(&#43;&#43;left &gt; right) break;
        }
        return matrix;
    }
```



### 498. 对角线遍历









## 二维数组变换



### 566. 重塑矩阵



笨方法，放入队列中，然后逐个按序取出。

```java
public int[][] matrixReshape(int[][] nums, int r, int c) {
        int[][] res = new int[r][c];
        if (nums.length == 0 || r * c != nums.length * nums[0].length)
            return nums;
        int count = 0;
        Queue &lt;Integer&gt; queue = new LinkedList &lt; &gt; ();
        for (int i = 0; i &lt; nums.length; i&#43;&#43;) {
            for (int j = 0; j &lt; nums[0].length; j&#43;&#43;) {
                queue.add(nums[i][j]);
            }
        }
        for (int i = 0; i &lt; r; i&#43;&#43;) {
            for (int j = 0; j &lt; c; j&#43;&#43;) {
                res[i][j] = queue.remove();
            }
        }
        return res;
    }
```



原地算法：

- $count$用于保存当前到第几个元素
- $count / c$用于计算当前是第几行，$count % c$用于计算当前是第几个。

```java
public int[][] matrixReshape(int[][] nums, int r, int c) {
        int[][] res = new int[r][c];
        if (nums.length == 0 || r * c != nums.length * nums[0].length)
            return nums;
        int count = 0;
        for (int i = 0; i &lt; nums.length; i&#43;&#43;) {
            for (int j = 0; j &lt; nums[0].length; j&#43;&#43;) {
                res[count / c][count % c] = nums[i][j];
                count&#43;&#43;;
            }
        }
        return res;
    }
```





---

> Author:   
> URL: http://localhost:1313/posts/%E7%AE%97%E6%B3%95/leetcode-array/  

