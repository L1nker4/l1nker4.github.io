# 排序算法总结


## 性质

### 稳定性

稳定性指相等的元素经过排序之后相对顺序是否发生了改变。



### 时间复杂度

对于基于比较的排序算法的时间复杂度，较好的性能是$O(nlogn)$，坏的性能是$O(n^2)$。一个排序算法的理想性能是$O(n)$，但是平均而言不可能达到。





## 选择排序

**Selection sort**是较为简单的一种排序算法，每次找到第$i$小的元素，然后将这个元素与数组的第$i$个位置上的元素交换。换句话说：每次找到未完成排序的数组中最小的值，然后将其与边界（已排序和未排序元素的边界）进行交换。

- 由于swap操作的存在，因此该算法是一种不稳定的排序算法
- 选择排序的最优、平均、最坏时间复杂度都是$O(n^2)$。两层for

Java：

```java
public static void selection(int[] arr){
	    int n = arr.length;
        for (int i = 1; i &lt; n; i&#43;&#43;) {
            int minIndex = i;
            for (int j = i &#43; 1; j &lt; n; j&#43;&#43;) {
                if (arr[j] &lt; arr[minIndex]){
                    minIndex = j;
                }
            }
            swap(arr, i, minIndex);
        }
    }
```



## 冒泡排序

**Bubble Sort**由于在算法的执行过程中，较小的元素像气泡一样慢慢浮到数组的顶端，故称为冒泡排序。

工作原理：每次检查相邻的两个元素，如果满足排序条件，则交换。经过$i$次扫描，数组末尾的$i$项必然是第$i$大的元素，因此冒泡排序最多需要扫描$n - 1$遍数组就能完成排序。

- 稳定的排序算法
- 序列完全有序时，冒泡排序只需遍历一次数组，不用执行任何交换操作，时间复杂度位$O (n) $
- 最坏情况下，需要进行$\frac{(n - 1)n}{2} $次交换操作。
- 平均时间复杂度为$O(n)$

```java
public static void bubbleSort(int[] arr) {
        boolean flag = true;
        while (flag) {
            flag = false;
            for (int i = 1; i &lt; arr.length -1; i&#43;&#43;) {
                if (arr[i] &gt; arr[i &#43; 1]) {
                    flag = true;
                    swap(arr, i, i &#43; 1);
                }
            }
        }
    }
```





## 插入排序

**Insertion Sort**：将待排序元素划分为已排序和未排序两部分，每次从未排序元素中选择一个插入到已排序的的元素中的正确位置。

案例：打扑克牌时，从桌上抓一张牌，按牌的大小插入正确的位置。

- 稳定的排序算法
- 最优的时间复杂度是$O(n)$
- 插入排序的最坏、平均时间复杂度为$O(n^2)$

```java
public static void insertionSort(int[] arr) {
        for (int i = 1; i &lt; arr.length; i&#43;&#43;) {
            for (int j = i - 1; j &gt;= 0; j--) {
                if (arr[j] &gt; arr[j &#43; 1]) {
                    swap(arr, j, j &#43; 1);
                }
            }
        }
    }
```



## 计数排序

**Counting Sort**：是一种线性时间的排序算法。

工作原理：使用一个额外的数组$C$，其中第$i$个元素是待排序数组$A$中值等于$i$的元素的个数，然后通过数组$C$来将$A$中的元素排到正确的位置。

分为三个步骤：

1. 计算每个数出现的次数
2. 求除每个数出现次数的前缀和
3. 利用出现次数的前缀和，从右至左计算每个数的排名



- 稳定的排序算法
- 时间复杂度为$O(n &#43; w)$，其中$w$代表待排序数据的值域大小。

```java
public static int[] countingSort(int[] A){
    //k是元素大小的上界
    int k = 100;
    int[] C = new int[k];
    int[] B = new int[A.length];
    for (int i = 0; i &lt; A.length; i&#43;&#43;) {
        C[A[i]]&#43;&#43;;
    }
    //求前缀和
    for (int i = 1; i &lt; k; i&#43;&#43;) {
        C[i] = C[i] &#43; C[i -1];
    }
    for (int j = A.length -1; j &gt;= 0; j--) {
        int a = A[j];
        B[C[a] - 1] = a;
        C[a]--;
    }
    return B;
}
```



## 归并排序

**merge sort**：是一种采用了分治思想的排序算法。

主要分为三个步骤：

- 将数组划分为两部分
- 递归地对两个子序列进行归并排序 
- 合并两个子序列



性质：

- 是一种稳定的排序算法
- 最优、最坏、平均时间复杂度均为$O(nlogn)$
- 归并排序的空间复杂度为$O(n)$

```java
public static void mergeSort(int[] arr, int low, int high) {
    int mid = (low &#43; high) / 2;
    if (low &lt; high) {
        //递归地对左右两边进行排序
        mergeSort(arr, low, mid);
        mergeSort(arr, mid &#43; 1, high);
        //合并
        merge(arr, low, mid, high);
    }
}

//merge实际上是将两个有序数组合并成一个有序数组
private static void merge(int[] arr, int low, int mid, int high) {
    //temp数组用于暂存合并的结果
    int[] temp = new int[high - low &#43; 1];
    //左半边的指针
    int i = low;
    //右半边的指针
    int j = mid &#43; 1;
    //合并后数组的指针
    int k = 0;

    //将记录由小到大地放进temp数组
    for (; i &lt;= mid &amp;&amp; j &lt;= high; k&#43;&#43;) {
        if (arr[i] &lt; arr[j])
            temp[k] = arr[i&#43;&#43;];
        else
            temp[k] = arr[j&#43;&#43;];
    }
    //接下来两个while循环是为了将剩余的（比另一边多出来的个数）放到temp数组中
    while (i &lt;= mid)
        temp[k&#43;&#43;] = arr[i&#43;&#43;];

    while (j &lt;= high)
        temp[k&#43;&#43;] = arr[j&#43;&#43;];

    //将temp数组中的元素写入到待排数组中
    for (int l = 0; l &lt; temp.length; l&#43;&#43;)
        arr[low &#43; l] = temp[l];
}
```



## 堆排序

**Heap Sort**：利用数据结构中的堆设计的一种排序算法。

工作原理：对所有待排序元素建堆， 利用最大堆（最小堆）的特性，依次取出堆顶元素，就可以得到排好序的数组。

当前节点的下为$i$时，它的父结点、左子节点、右子节点的获取方式如下：

```
//向下舍入
parent = floor((i -  1) /2);
leftChild = 2* i &#43; 1;
rightChild = 2 * i &#43; 2;
```



- 不稳定的排序算法
- 最优、平均、最坏时间复杂度均为$O(nlogn)$。



```java
public static void heapSort(int[] arr) {
        //生成大根堆
        int len = arr.length;
        for (int i = len / 2 - 1; i &gt;= 0; i--) {
            heapify(arr, i, len);
        }
        for (int j = arr.length -1; j &gt; 0; j--) {
            swap(arr, 0, j);
            heapify(arr, 0, j);
        }
    }

    private static void heapify(int[] arr, int i, int length) {
        int temp = arr[i];
        for (int k = i * 2 &#43; 1; k &lt; length; k = k * 2 &#43; 1) {
            //如果左子节点小于右子节点，k指向右子节点
            if (k &#43; 1 &lt; length &amp;&amp; arr[k] &lt; arr[k &#43; 1]) {
                k&#43;&#43;;
            }
            if (arr[k] &gt; temp){
                //如果子节点大于父结点，将子节点值赋值给父结点
                arr[i] = arr[k];
                i = k;
            }else {
                break;
            }
        }
        arr[i] = temp;
    }
```



## 快速排序

**Quick Sort**：简称为快排，也成为分区交换排序。是一种广泛运用的排序算法。



基本原理：通过分治思想实现排序。

1. 选取基准值，以基准值为界，将比它大的和比它小的分别放在两边。
2. 对子序列进行递归快排，直至序列为空或者只有一个元素。



```java
public static void quickSort(int[] arr, int start, int end) {
        if (start &gt;= end) {
            return;
        }
        int pivotIndex = partition(arr, start, end);
        quickSort(arr, start, pivotIndex - 1);
        quickSort(arr, pivotIndex &#43; 1, end);
    }

    /**
     * 分治
     *
     * @param arr
     * @param start
     * @param end
     * @return
     */
    private static int partition(int[] arr, int start, int end) {
        //将第一个元素作为基准元素
        int pivot = arr[start];
        int left = start;
        int right = end;
        while (left != right) {
            //控制right并左移
            while (left &lt; right &amp;&amp; arr[right] &gt; pivot) {
                right--;
            }
            //控制left并右移
            while (left &lt; right &amp;&amp; arr[left] &lt;= pivot) {
                left&#43;&#43;;
            }
            if (left &lt; right) {
                swap(arr, left, right);
            }
        }
        //pivot和指针重合点交换
        arr[start] = arr[left];
        arr[left] = pivot;
        return left;
    }
```



## 希尔排序

**Shell Sort**：最小增量排序，是插入排序的一种改进版本。

工作原理：对不相邻的记录进行比较和移动。

1. 将待排序数组分成若干个子序列（每个子序列的元素在原始数组中间距相同）
2. 对这些子序列进行插入排序
3. 减少每个子序列中元素之间的间距，重复上述过程直至间距减少为1。



- 不稳定的排序算法
- 最优时间复杂度为$O(n)$。
- 平均时间复杂度和最坏时间复杂度与间距序列的选取有关，已知最好的最坏时间复杂度为$O(nlog^2n)$。
- 空间复杂度为$O(n)$。



```java
public static void shellSort(int[] arr, int length) {
        int h = 1;
        while (h &lt; length / 3) {
            h = 3 * h &#43; 1;
        }
        while (h &gt;= 1) {
            for (int i = h; i &lt; length; i&#43;&#43;) {
                for (int j = i; j &gt;= h &amp;&amp; arr[j] &lt; arr[j - h]; j -= h) {
                    swap(arr, j, j - h);
                }
            }
            h = h / 3;
        }
}
```





## 基数排序

**Radix Sort**：是一种非比较型整数排序算法。

工作原理：将待排序的元素拆分为$k$个关键字，然后先对第$k$个关键字进行稳定排序，再对第$k - 1$个关键字进行稳定排序，……最后对第一个关键字进行稳定排序。

大白话：先根据个位排序，再根据十位排序，再按照百位排序……

基数排序中需要借助一种**稳定算法**完成内层对关键字的排序。



基数排序根据两种不同的排序顺序，可以分为：

- LSD（Least significant digital）：从低位开始
- MSD（Most significant digital）：从高位开始



性质：

- 基数排序是一种稳定的排序算法。
- 如果每个关键字的值域不大，可以使用**计数排序**作为内层排序，此时复杂度为$O(kn &#43; \sum\limits_{i=1}^{k}w_i)$，其中$w_i$为第$i$个关键字值域的大小，如果关键字值域很大，就可以使用基于比较的$O(nklogn)$的排序。
- 空间复杂度$O(k &#43; n)$





## 桶排序

**Bucket Sort**：是排序算法的一种，适用于待排序数据值域较大但分布较为均匀的情况。

步骤：

1. 设置一个定量的数组当作空桶
2. 遍历序列，将元素一个个的放入对应的桶中
3. 对每个不是空的桶中元素取出来，按序放入序列。



性质：

- 如果使用稳定的内层排序，并且将元素插入桶中时不改变元素的相对顺序，那么桶排序是一种稳定的排序算法。
- 桶排序的平均时间复杂度为$O(n &#43; n^2/k &#43; k)$，（将值域平均分成n块 &#43; 排序 &#43; 重新合并元素），当$k \approx n $时为$O(n)$。
- 最坏时间复杂度为$O(n^2)$。



```java
private static int indexFor(int a, int min, int step) {
        return (a - min) / step;
    }

    public static void bucketSort(int[] arr) {

        int max = arr[0], min = arr[0];
        for (int a : arr) {
            if (max &lt; a)
                max = a;
            if (min &gt; a)
                min = a;
        }
        int bucketNum = max / 10 - min / 10 &#43; 1;
        List buckList = new ArrayList&lt;List&lt;Integer&gt;&gt;();
        // create bucket
        for (int i = 1; i &lt;= bucketNum; i&#43;&#43;) {
            buckList.add(new ArrayList&lt;Integer&gt;());
        }
        // push into the bucket
        for (int i = 0; i &lt; arr.length; i&#43;&#43;) {
            int index = indexFor(arr[i], min, 10);
            ((ArrayList&lt;Integer&gt;) buckList.get(index)).add(arr[i]);
        }
        ArrayList&lt;Integer&gt; bucket = null;
        int index = 0;
        for (int i = 0; i &lt; bucketNum; i&#43;&#43;) {
            bucket = (ArrayList&lt;Integer&gt;) buckList.get(i);
            insertSort(bucket);
            for (int k : bucket) {
                arr[index&#43;&#43;] = k;
            }
        }

    }

    private static void insertSort(List&lt;Integer&gt; bucket) {
        for (int i = 1; i &lt; bucket.size(); i&#43;&#43;) {
            int temp = bucket.get(i);
            int j = i - 1;
            for (; j &gt;= 0 &amp;&amp; bucket.get(j) &gt; temp; j--) {
                bucket.set(j &#43; 1, bucket.get(j));
            }
            bucket.set(j &#43; 1, temp);
        }
    }
```







## 参考

OI WIKI

Wikipedia



























---

> Author:   
> URL: http://localhost:1313/posts/%E7%AE%97%E6%B3%95/sort-algorithm/  

