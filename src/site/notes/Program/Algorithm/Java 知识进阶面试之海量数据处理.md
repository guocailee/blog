---
{"dg-publish":true,"permalink":"/Program/Algorithm/Java 知识进阶面试之海量数据处理/","noteIcon":"","created":"2024-05-22T16:17:54.115+08:00"}
---

#Algorithm 
## 1.  如何从大量的 URL 中找出相同的 URL？

### 题目描述

给定 a、b 两个文件，各存放 50 亿个 URL，每个 URL 各占 64B，内存限制是 4G。请找出 a、b 两个文件共同的 URL。

### 解答思路

####  分治策略

每个 URL 占 64B，那么 50 亿个 URL 占用的空间大小约为 320GB。

```
5, 000, 000, 000 _ 64B ≈ 5GB _ 64 = 320GB
```

由于内存大小只有 4G，因此，我们不可能一次性把所有 URL 加载到内存中处理。对于这种类型的题目，一般采用分治策略，即：把一个文件中的 URL 按照某个特征划分为多个小文件，使得每个小文件大小不超过 4G，这样就可以把这个小文件读到内存中进行处理了。

思路如下：

首先遍历文件 a，对遍历到的 URL 求 hash(URL) % 1000 ，根据计算结果把遍历到的 URL 存储到 a0, a1, a2, …, a999，这样每个大小约为 300MB。使用同样的方法遍历文件 b，把文件 b 中的 URL 分别存储到文件 b0, b1, b2, …, b999 中。这样处理过后，所有可能相同的 URL 都在对应的小文件中，即 a0 对应 b0, …, a999 对应 b999，不对应的小文件不可能有相同的 URL。那么接下来，我们只需要求出这 1000 对小文件中相同的 URL 就好了。

接着遍历 ai( i∈\[0,999\] )，把 URL 存储到一个 HashSet 集合中。然后遍历 bi 中每个 URL，看在 HashSet 集合中是否存在，若存在，说明这就是共同的 URL，可以把这个 URL 保存到一个单独的文件中。

####  前缀树

一般而言，URL 的长度差距不会不大，而且前面几个字符，绝大部分相同。这种情况下，非常适合使用[[Program/Algorithm/Leetcode 题解 - 树#Trie\|Leetcode 题解 - 树#Trie]] 这种数据结构来进行存储，降低存储成本的同时，提高查询效率。

#### 方法总结
--------

*   分治策略：分而治之，进行哈希取余；对每个子文件进行 HashSet 统计。
    
*   前缀树：利用字符串的公共前缀来降低存储成本，提高查询效率。
    
## 2.  如何从大文件查找频数最高的词？

###  题目描述
--------

有一个 1GB 大小的文件，文件里每一行是一个词，每个词的大小不超过 16B，内存大小限制是 1MB，要求返回频数最高的 100 个词(Top 100)。

### 解答思路

> 由于内存限制，我们依然无法直接将大文件的所有词一次读到内存中。
因此，同样可以采用分治策略，把一个大文件分解成多个小文件，保证每个文件的大小小于 1MB，进而直接将单个小文件读取到内存中进行处理。

#### 思路如下
-----

首先遍历大文件，对遍历到的每个词 x，执行 hash(x) % 5000 ，将结果为 i 的词存放到文件 ai 中。遍历结束后，我们可以得到 5000 个小文件。每个小文件的大小为 200KB 左右。如果有的小文件大小仍然超过 1MB，则采用同样的方式继续进行分解。

接着统计每个小文件中出现频数最高的 100 个词。最简单的方式是使用 HashMap 来实现。其中 key 为词，value 为该词出现的频率。具体方法是：对于遍历到的词 x，如果在 map 中不存在，则执行 map.put(x, 1) ；若存在，则执行 map.put(x, map.get(x)+1) ，将该词频数加 1。

上面我们统计了每个小文件单词出现的频数。接下来，我们可以通过维护一个小顶堆来找出所有词中出现频数最高的 100 个。具体方法是：依次遍历每个小文件，构建一个小顶堆，堆大小为 100。如果遍历到的词的出现次数大于堆顶词的出现次数，则用新词替换堆顶的词，然后重新调整为小顶堆，遍历结束后，小顶堆上的词就是出现频数最高的 100 个词。

### 方法总结
--------

*   分而治之，进行哈希取余；
    
*   使用 HashMap 统计频数；
    
*   求解最大的 TopN 个，用小顶堆；求解最小的 TopN 个，用大顶堆。
    
## 3. 如何从超大日志中查找访问最多的IP
### 题目描述 
--------

现有海量日志数据保存在一个超大文件中，该文件无法直接读入内存，要求从中提取某天访问百度次数最多的那个 IP。

### 解答思路
--------

这道题只关心某一天访问百度最多的 IP，因此，可以首先对文件进行一次遍历，把这一天访问百度 IP 的相关信息记录到一个单独的大文件中。接下来采用的方法与上一题一样，大致就是先对 IP 进行哈希映射，接着使用 HashMap 统计重复 IP 的次数，最后计算出重复次数最多的 IP。

注：这里只需要找出出现次数最多的 IP，可以不必使用堆，直接用一个变量 max 即可。

### 方法总结
--------

*   分而治之，进行哈希取余；
    
*   使用 HashMap 统计频数；
    
*   求解最大的 TopN 个，用小顶堆；求解最小的 TopN 个，用大顶堆。
    
## 4. 在 2.5 亿个整数中找出不重复的整数
### 题目描述
--------

在 2.5 亿个整数中找出不重复的整数。注意：内存不足以容纳这 2.5 亿个整数。

### 解答思路
----

#### 方法一：分治法

与前面的题目方法类似，先将 2.5 亿个数划分到多个小文件，用 HashSet/HashMap 找出每个小文件中不重复的整数，再合并每个子结果，即为最终结果。

#### 方法二：位图法

位图，就是用一个或多个 bit 来标记某个元素对应的值，而键就是该元素。采用位作为单位来存储数据，可以大大节省存储空间。

位图通过使用位数组来表示某些元素是否存在。它可以用于快速查找，判重，排序等。不是很清楚？我先举个小例子。

假设我们要对 \[0,7\] 中的 5 个元素 (6, 4, 2, 1, 5) 进行排序，可以采用位图法。

0~7 范围总共有 8 个数，只需要 8bit，即 1 个字节。首先将每个位都置 0：

```properties
0 0 0 0 0 0 0 0
```

然后遍历 5 个元素，首先遇到 6，那么将下标为 6 的位的 0 置为 1；接着遇到 4，把下标为 4 的位 的 0 置为 1：

```properties
0 0 0 0 1 0 1 0
```

依次遍历，结束后，位数组是这样的：

```properties
0 1 1 0 1 1 1 0
```

每个为 1 的位，它的下标都表示了一个数：

```bash
for i in range(8)
    if bits[i] == 1:
        print(i)
```

这样我们其实就已经实现了排序。

对于整数相关的算法的求解，位图法是一种非常实用的算法。假设 int 整数占用 4B，即 32bit，那么我们可以表示的整数的个数为 232。

那么对于这道题，我们用 2 个 bit 来表示各个数字的状态：

00 表示这个数字没出现过；01 表示这个数字出现过一次（即为题目所找的不重复整数）；10 表示这个数字出现了多次。那么这 232 个整数，总共所需内存为 232*2b=1GB。因此，当可用内存超过 1GB 时，可以采用位图法。假设内存满足位图法需求，进行下面的操作：

遍历 2.5 亿个整数，查看位图中对应的位，如果是 00，则变为 01，如果是 01 则变为 10，如果是 10 则保持不变。遍历结束后，查看位图，把对应位是 01 的整数输出即可。

当然，本题中特别说明：内存不足以容纳这 2.5 亿个整数，2.5 亿个整数的内存大小为：2.5e8/1024/1024/1024 * 4=3.72GB， 如果内存大于 1GB，是可以通过位图法解决的。

### 方法总结
--------

判断数字是否重复的问题，位图法是一种非常高效的方法，当然前提是：内存要满足位图法所需要的存储空间。

## 5.  判断某数是否在40亿个不重复没排序中
### 题目描述
--------

给定 40 亿个不重复的没排过序的 unsigned int 型整数，然后再给定一个数，如何快速判断这个数是否在这 40 亿个整数当中？

### 解答思路
--------

#### 方法一：分治法

依然可以用分治法解决，方法与前面类似，就不再次赘述了。

#### 方法二：位图法

由于 unsigned int 数字的范围是 \[0, 1 « 32)，我们用 1«32=4,294,967,296 个 bit 来表示每个数字。初始位均为 0，那么总共需要内存：4,294,967,296b≈512M。

我们读取这 40 亿个整数，将对应的 bit 设置为 1。接着读取要查询的数，查看相应位是否为 1，如果为 1 表示存在，如果为 0 表示不存在。

### 方法总结
--------

判断数字是否存在、判断数字是否重复的问题，位图法是一种非常高效的方法。
## 6. 统计最热门的查询字符串
### 题目描述 
--------

搜索引擎会通过日志文件把用户每次检索使用的所有查询串都记录下来，每个查询串的长度不超过 255 字节。

假设目前有 1000w 个记录（这些查询串的重复度比较高，虽然总数是 1000w，但如果除去重复后，则不超过 300w 个）。请统计最热门的 10 个查询串，要求使用的内存不能超过 1G。（一个查询串的重复度越高，说明查询它的用户越多，也就越热门。）

### 解答思路
--------

每个查询串最长为 255B，1000w 个串需要占用 约 2.55G 内存，因此，我们无法将所有字符串全部读入到内存中处理。

#### 方法一：分治法

分治法依然是一个非常实用的方法。

划分为多个小文件，保证单个小文件中的字符串能被直接加载到内存中处理，然后求出每个文件中出现次数最多的 10 个字符串；最后通过一个小顶堆统计出所有文件中出现最多的 10 个字符串。

方法可行，但不是最好，下面介绍其他方法。

####  方法二：HashMap 法

虽然字符串总数比较多，但去重后不超过 300w，因此，可以考虑把所有字符串及出现次数保存在一个 HashMap 中，所占用的空间为 300w*(255+4)≈777M（其中，4 表示整数占用的 4 个字节）。由此可见，1G 的内存空间完全够用。

思路如下：

首先，遍历字符串，若不在 map 中，直接存入 map，value 记为 1；若在 map 中，则把对应的 value 加 1，这一步时间复杂度 O(N) 。

接着遍历 map，构建一个 10 个元素的小顶堆，若遍历到的字符串的出现次数大于堆顶字符串的出现次数，则进行替换，并将堆调整为小顶堆。

遍历结束后，堆中 10 个字符串就是出现次数最多的字符串。这一步时间复杂度 O(Nlog10) 。

#### 方法三：前缀树法

方法二使用了 HashMap 来统计次数，当这些字符串有大量相同前缀时，可以考虑使用前缀树来统计字符串出现的次数，树的结点保存字符串出现次数，0 表示没有出现。

思路如下：

在遍历字符串时，在前缀树中查找，如果找到，则把结点中保存的字符串次数加 1，否则为这个字符串构建新结点，构建完成后把叶子结点中字符串的出现次数置为 1。

最后依然使用小顶堆来对字符串的出现次数进行排序。

### 方法总结
--------

前缀树经常被用来统计字符串的出现次数。它的另外一个大的用途是字符串查找，判断是否有重复的字符串等。
## 7.  统计不同号码的个数
### 题目描述
--------

已知某个文件内包含一些电话号码，每个号码为 8 位数字，统计不同号码的个数。

### 解答思路
--------

这道题本质还是求解数据重复的问题，对于这类问题，一般首先考虑位图法。

对于本题，8 位电话号码可以表示的号码个数为 108 个，即 1 亿个。我们每个号码用一个 bit 来表示，则总共需要 1 亿个 bit，内存占用约 12M。

思路如下：

申请一个位图数组，长度为 1 亿，初始化为 0。然后遍历所有电话号码，把号码对应的位图中的位置置为 1。遍历完成后，如果 bit 为 1，则表示这个电话号码在文件中存在，否则不存在。bit 值为 1 的数量即为 不同电话号码的个数。

#### 方法总结
--------

求解数据重复问题，记得考虑位图法。

### 题目描述
--------

从 5 亿个数中找出中位数。数据排序后，位置在最中间的数就是中位数。当样本数为奇数时，中位数为 第 (N+1)/2 个数；当样本数为偶数时，中位数为 第 N/2 个数与第 1+N/2 个数的均值。

**解答思路**
--------

如果这道题没有内存大小限制，则可以把所有数读到内存中排序后找出中位数。但是最好的排序算法的时间复杂度都为 O(NlogN) 。这里使用其他方法。

### **方法一：双堆法**

维护两个堆，一个大顶堆，一个小顶堆。大顶堆中最大的数小于等于小顶堆中最小的数；保证这两个堆中的元素个数的差不超过 1。

若数据总数为偶数，当这两个堆建好之后，中位数就是这两个堆顶元素的平均值。当数据总数为奇数时，根据两个堆的大小，中位数一定在数据多的堆的堆顶。

```cpp
class MedianFinder {

    private PriorityQueue<Integer> maxHeap;
    private PriorityQueue<Integer> minHeap;

    
    public MedianFinder() {
        maxHeap = new PriorityQueue<>(Comparator.reverseOrder());
        minHeap = new PriorityQueue<>(Integer::compareTo);
    }

    public void addNum(int num) {
        if (maxHeap.isEmpty() || maxHeap.peek() > num) {
            maxHeap.offer(num);
        } else {
            minHeap.offer(num);
        }

        int size1 = maxHeap.size();
        int size2 = minHeap.size();
        if (size1 - size2 > 1) {
            minHeap.offer(maxHeap.poll());
        } else if (size2 - size1 > 1) {
            maxHeap.offer(minHeap.poll());
        }
    }

    public double findMedian() {
        int size1 = maxHeap.size();
        int size2 = minHeap.size();

        return size1 == size2
            ? (maxHeap.peek() + minHeap.peek()) * 1.0 / 2
            : (size1 > size2 ? maxHeap.peek() : minHeap.peek());
    }
}
```

以上这种方法，需要把所有数据都加载到内存中。当数据量很大时，就不能这样了，因此，这种方法适用于数据量较小的情况。5 亿个数，每个数字占用 4B，总共需要 2G 内存。如果可用内存不足 2G，就不能使用这种方法了，下面介绍另一种方法。

### **分治法**

分治法的思想是把一个大的问题逐渐转换为规模较小的问题来求解。

对于这道题，顺序读取这 5 亿个数字，对于读取到的数字 num，如果它对应的二进制中最高位为 1，则把这个数字写到 f1 中，否则写入 f0 中。通过这一步，可以把这 5 亿个数划分为两部分，而且 f0 中的数都大于 f1 中的数（最高位是符号位）。

划分之后，可以非常容易地知道中位数是在 f0 还是 f1 中。假设 f1 中有 1 亿个数，那么中位数一定在 f0 中，且是在 f0 中，从小到大排列的第 1.5 亿个数与它后面的一个数的平均值。

提示，5 亿数的中位数是第 2.5 亿与右边相邻一个数求平均值。若 f1 有一亿个数，那么中位数就是 f0 中从第 1.5 亿个数开始的两个数求得的平均值。

对于 f0 可以用次高位的二进制继续将文件一分为二，如此划分下去，直到划分后的文件可以被加载到内存中，把数据加载到内存中以后直接排序，找出中位数。

注意，当数据总数为偶数，如果划分后两个文件中的数据有相同个数，那么中位数就是数据较小的文件中的最大值与数据较大的文件中的最小值的平均值。

**题目描述**
--------

有 10 个文件，每个文件大小为 1G，每个文件的每一行存放的都是用户的 query，每个文件的 query 都可能重复。要求按照 query 的频度排序。

**解答思路**
--------

如果 query 的重复度比较大，可以考虑一次性把所有 query 读入内存中处理；如果 query 的重复率不高，那么可用内存不足以容纳所有的 query，这时候就需要采用分治法或其他的方法来解决。

### **方法一：HashMap 法**

如果 query 重复率高，说明不同 query 总数比较小，可以考虑把所有的 query 都加载到内存中的 HashMap 中。接着就可以按照 query 出现的次数进行排序。

### **方法二：分治法**

分治法需要根据数据量大小以及可用内存的大小来确定问题划分的规模。对于这道题，可以顺序遍历 10 个文件中的 query，通过 Hash 函数 hash(query) % 10 把这些 query 划分到 10 个小文件中。

之后对每个小文件使用 HashMap 统计 query 出现次数，根据次数排序并写入到另外一个单独文件中。

接着对所有文件按照 query 的次数进行排序，这里可以使用归并排序（由于无法把所有 query 都读入内存，因此需要使用外排序）。

**方法总结**
--------

*   内存若够，直接读入进行排序；
    
*   内存不够，先划分为小文件，小文件排好序后，整理使用外排序进行归并。
    

**题目描述**
--------

有 20 个数组，每个数组有 500 个元素，并且有序排列。如何在这 20*500 个数中找出前 500 的数？

**解答思路**
--------

对于 TopK 问题，最常用的方法是使用堆排序。对本题而言，假设数组降序排列，可以采用以下方法：

首先建立大顶堆，堆的大小为数组的个数，即为 20，把每个数组最大的值存到堆中。

接着删除堆顶元素，保存到另一个大小为 500 的数组中，然后向大顶堆插入删除的元素所在数组的下一个元素。

重复上面的步骤，直到删除完第 500 个元素，也即找出了最大的前 500 个数。

为了在堆中取出一个数据后，能知道它是从哪个数组中取出的，从而可以从这个数组中取下一个值，可以把数组的指针存放到堆中，对这个指针提供比较大小的方法。

```cpp
import lombok.Data;

import java.util.Arrays;
import java.util.PriorityQueue;




@Data
public class DataWithSource implements Comparable<DataWithSource> {
    


    private int value;

    


    private int source;

    


    private int index;

    public DataWithSource(int value, int source, int index) {
        this.value = value;
        this.source = source;
        this.index = index;
    }

    




    @Override
    public int compareTo(DataWithSource o) {
        return Integer.compare(o.getValue(), this.value);
    }
}

class Test {
    public static int[] getTop(int[][] data) {
        int rowSize = data.length;
        int columnSize = data[0].length;

        
        int[] result = new int[columnSize];

        PriorityQueue<DataWithSource> maxHeap = new PriorityQueue<>();
        for (int i = 0; i < rowSize; ++i) {
            
            DataWithSource d = new DataWithSource(data[i][0], i, 0);
            maxHeap.add(d);
        }

        int num = 0;
        while (num < columnSize) {
            
            DataWithSource d = maxHeap.poll();
            result[num++] = d.getValue();
            if (num >= columnSize) {
                break;
            }

            d.setValue(data[d.getSource()][d.getIndex() + 1]);
            d.setIndex(d.getIndex() + 1);
            maxHeap.add(d);
        }
        return result;

    }

    public static void main(String[] args) {
        int[][] data = {
                {29, 17, 14, 2, 1},
                {19, 17, 16, 15, 6},
                {30, 25, 20, 14, 5},
        };

        int[] top = getTop(data);
        System.out.println(Arrays.toString(top)); 
    }
}
```

对于海量数据到处理经常会涉及到 topK 问题。在设计数据结构和算法的时候，主要需要考虑的应该是当前算法（包括数据结构）跟给定情境（比如数据量级、数据类型）的适配程度，和当前问题最核心的瓶颈（如降低时间复杂度，还是降低空间复杂度）是什么。

首先，我们来举几个常见的 topK 问题的例子：

给定 100 个 int 数字，在其中找出最大的 10 个；

给定 10 亿个 int 数字，在其中找出最大的 10 个（这 10 个数字可以无序）；

给定 10 亿个 int 数字，在其中找出最大的 10 个（这 10 个数字依次排序）；

给定 10 亿个不重复的 int 数字，在其中找出最大的 10 个；

给定 10 个数组，每个数组中有 1 亿个 int 数字，在其中找出最大的 10 个；

给定 10 亿个 string 类型的数字，在其中找出最大的 10 个（仅需要查 1 次）；

给定 10 亿个 string 类型的数字，在其中找出最大的 k 个（需要反复多次查询，其中 k 是一个随机数字）。

上面这些问题看起来很相似，但是解决的方式却千差万别。

稍有不慎，就可能使得 topK 问题成为系统的瓶颈。不过也不用太担心，接下来我会总结几种常见的解决思路，遇到问题的时候，大家把这些基础思路融会贯通并且杂糅组合，即可做到见招拆招。

**1\. 堆排序法**
------------

这里说的是堆排序法，而不是快排或者希尔排序。虽然理论时间复杂度都是 O(nlogn)，但是堆排在做 topK 的时候有一个优势，就是可以维护一个仅包含 k 个数字的小顶堆（想清楚，为啥是小顶堆哦），当新加入的数字大于堆顶数字的时候，将堆顶元素剔除，并加入新的数字。

用 C++ 来说明，堆在 stl 中是 priority_queue（不是 set）。

```cpp
int main() {
    const int topK = 3;
    vector<int> vec = {4,1,5,8,7,2,3,0,6,9};
    priority_queue<int, vector<int>, greater<>> pq;    
    for (const auto& x : vec) {
        pq.push(x);
        if (pq.size() > topK) {
            
            pq.pop();
        }
    }

    while (!pq.empty()) {
        cout << pq.top() << endl;    
        pq.pop();
    }

    return 0;
}
```

**2\. 类似快排法**
-------------

快排大家都知道，针对 topK 问题，可以对快排进行改进。仅对部分数据进行递归计算。

比如，在 100 个数字中，找最大的 10 个，第一次循环的时候，povit 被移动到了 80 的位置，则接下来仅需要在后面的 20 个数字中找最大的 10 个即可。

这样做的优势是，理论最优时间复杂度可以达到 O(n)，不过平均时间复杂度还是 O(nlogn)。需要说明的是，通过这种方式，找出来的最大的 k 个数字之间，是无序的。

```cpp
int partition(vector<int>& arr, int begin, int end) {
    int left = begin;
    int right = end;
    int povit = arr[begin];

    while (left < right) {
        while (left < right && arr[right] >= povit) {right--;}
        while (left < right && arr[left] <= povit) {left++;}
        if (left < right) {swap(arr[left], arr[right]);}
    }

    swap(arr[begin], arr[left]);
    return left;
}

void partSort(vector<int>& arr, int begin, int end, int target) {
    if (begin >= end) {
        return;
    }

    int povit = partition(arr, begin, end);
    if (target < povit) {
        partSort(arr, begin, povit - 1, target);
    } else if (target > povit) {
        partSort(arr, povit + 1, end, target);
    }
}

vector<int> getMaxNumbers(vector<int>& arr, int k) {
    int size = (int)arr.size();
    
    int target = size - k;
    partSort(arr, 0, size - 1, target);
    vector<int> ret(arr.end() - k, arr.end());
    return ret;
}

int main() {
    vector<int> vec = {4,1,5,8,7,2,3,0,6,9};
    auto ret = getMaxNumbers(vec, 3);

    for (auto x : ret) {
        cout << x << endl;    
    }

    return 0;
}
```

**3\. 使用 bitmap**
-----------------

有时候 topK 问题会遇到数据量过大，内存无法全部加载。这个时候，可以考虑将数据存放至 bitmap 中，方便查询。

比如，给出 10 个 int 类型的数据，分别是【13，12，11，1，2，3，4，5，6，7】，int 类型的数据每个占据 4 个字节，那这个数组就占据了 40 个字节。

现在，把它们放到一个 16 个长度 bool 的 bitmap 中，结果就是【0，1，1，1，1，1，1，1，0，0，0，1，1，1，0，0】，在将空间占用降低至 4 字节的同时，也可以很方便的看出，最大的 3 个数字，分别是 11，12 和 13。

需要说明的是，bitmap 结合跳表一起使用往往有奇效。比如以上数据还可以记录成：从第 1 位开始，有连续 7 个 1；从第 11 位开始，有连续 3 个 1。这样做，空间复杂度又得到了进一步的降低。

这种做法的优势，当然是降低了空间复杂度。不过需要注意一点，bitmap 比较适合不重复且有范围（比如，数据均在 0 ～ 10 亿之间）的数据的查询。至于有重复数据的情况，可以考虑与 hash 等结构的混用。

**4\. 使用 hash**
---------------

如果遇到了查询 string 类型数据的大小，可以考虑 hash 方法。

举个例子，10 个 string 数字【”1001”，”23”，”1002”，”3003”，”2001”，”1111”，”65”，”834”，”5”，”987”】找最大的 3 个。我们先通过长度进行 hash，得到长度最大为 4，且有 5 个长度为 4 的 string。接下来再通过最高位值做 hash，发现有 1 个最高位为”3”的，1 个为”2”的，3 个为”1”的。接下来，可以通过再设计 hash 函数，或者是循环的方式，在 3 个最高位为”1”的 string 中找到最大的一个，即可找到 3 个最值大的数据。

这种方法比较适合网址或者电话号码的查询。缺点就是如果需要多次查询的话，需要多次计算 hash，并且需要根据实际情况设计多个 hash 函数。

**5\. 字典树**
-----------

字典树（trie）的具体结构和查询方式，不在这里赘述了，自行百度一下就有很多。这里主要说一下优缺点。

字典树的思想，还是通过前期建立索引信息，后期可以反复多次查询，并且后期增删数据也很方便。比较适合于需要反复多次查询的情况。

比如，反复多次查询字符序（例如：z>y>…>b>a）最大的 k 个 url 这种，使用字典树把数据存储一遍，就非常适合。既减少了空间复杂度，也加速了查询效率。

**6\. 混合查询**
------------

以上几种方法，都是比较独立的方法。其实，在实际工作中，遇到更多的问题还是混合问题，这就需要我们对相关的内容，融会贯通并且做到活学活用。

我举个例子：我们的分布式服务跑在 10 台不同机器上，每台机器上部署的服务均被请求 10000 次，并且记录了个这 10000 次请求的耗时（耗时值为 int 数据），找出这 10*10000 次请求中，从高到低的找出耗时最大的 50 个。看看这个问题，很现实吧。我们试着用上面介绍的方法，组合一下来求解。

### **方法一**

首先，对每台机器上的 10000 个做类似快排，找出每台机器上 top50 的耗时信息。此时，单机上的这 50 条数据是无序的。

然后，再将 10 台机器上的 50 条数据（共 500 条）放到一起，再做一次类似快排，找到最大的 50 个（此时应该这 50 个应该是无序的）。

最后，对这 50 个数据做快排，从而得到最终结果。

### **方法二**

首先通过堆排，分别找出 10 台机器上耗时最高的 50 个数据，此时的这 50 个数据，已经是从大到小有序的了。

然后，我们依次取出 10 台机器中，耗时最高的 5 条放入小顶堆中。

最后，遍历 10 台机器上的数据，每台机器从第 6 个数据开始往下循环，如果这个值比堆顶的数据大，则抛掉堆顶数据并且把它加入，继续用下一个值进行同样比较。如果这个值比堆顶的值小，则结束当前循环，并且在下一台机器上做同样操作。

以上我介绍了两种方法，并不是为了说明哪种方法更好，或者时间复杂度更低。而是想说同样的事情有多种不同的解决方法，而且随着数据量的增加，可能会需要更多组合形式。在这个领域，数据决定了数据结构，数据结构决定了算法。

没有最好的方法，只有不断找寻更好的方法的程序员。适合的，才会是最好的。

*   https://github.com/doocs/advanced-java/blob/main/docs/big-data/topk-problems-and-solutions.md
    
*   https://github.com/doocs/advanced-java/blob/main/docs/big-data/find-rank-top-500-numbers.md
    
*   https://github.com/doocs/advanced-java/blob/main/docs/big-data/sort-the-query-strings-by-counts.md
    
*   https://github.com/doocs/advanced-java/blob/main/docs/big-data/find-mid-value-in-500-millions.md
    
*   https://github.com/doocs/advanced-java/blob/main/docs/big-data/count-different-phone-numbers.md
    
*   https://github.com/doocs/advanced-java/blob/main/docs/big-data/find-hotest-query-string.md
    
*   https://github.com/doocs/advanced-java/blob/main/docs/big-data/find-a-number-if-exists.md
    
*   https://github.com/doocs/advanced-java/blob/main/docs/big-data/find-no-repeat-number.md
    
*   https://github.com/doocs/advanced-java/blob/main/docs/big-data/find-top-1-ip.md
    
*   https://github.com/doocs/advanced-java/blob/main/docs/big-data/find-top-100-words.md
    
*   https://github.com/doocs/advanced-java/blob/main/docs/big-data/find-common-urls.md
    

> 转自：老马啸西风，
> 
> 链接：houbb.github.io/2022/12/14/advanced-java-interview-16-big-data-handle

\- EOF -

推荐阅读  点击标题可跳转

1、[刨根问底 Kafka，面试过程真好使](http://mp.weixin.qq.com/s?__biz=MjM5NzMyMjAwMA==&mid=2651521969&idx=1&sn=2bb4a0c1a7d6e307f73f21d285eb8f88&chksm=bd246fce8a53e6d8cf0530b06cef6ce45dae7bae0a0e8c015934e575e3b94d140767170ef2ea&scene=21#wechat_redirect)

2、[JVM 面试题第二版，据说看完要一小时](http://mp.weixin.qq.com/s?__biz=MjM5NzMyMjAwMA==&mid=2651504505&idx=1&sn=4c12c850bb2e7a9423d6af9b9c5966c3&chksm=bd25d3068a525a10a51859151dce4611e5b6d74bafa280b7bf2d42070a24b7319238975349fb&scene=21#wechat_redirect)

3、[](http://mp.weixin.qq.com/s?__biz=MjM5NzMyMjAwMA==&mid=2651521969&idx=1&sn=2bb4a0c1a7d6e307f73f21d285eb8f88&chksm=bd246fce8a53e6d8cf0530b06cef6ce45dae7bae0a0e8c015934e575e3b94d140767170ef2ea&scene=21#wechat_redirect)[面试题详解：用 Redis 实现分布式锁的血泪史](http://mp.weixin.qq.com/s?__biz=MjM5NzMyMjAwMA==&mid=2651513117&idx=1&sn=aff1b634f8447c9fd6861c65b03ec745&chksm=bd258d628a5204740d4205b03996776623f4d610c9c66ccb393fb09d8a80583a050be68dad34&scene=21#wechat_redirect)
