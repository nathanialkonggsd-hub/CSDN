@[TOC](leetcode数据结构与算法1~4)

# 从语法到算法

语言的尽头是算法。学完了 C 语言的语法，只能停留在“能写出代码”的阶段，遇到稍微复杂点的数据处理就无从下手，或是只能一味的for暴力。

所以，我切换了实战模式。在接下来的数据结构篇，不讲干巴巴的理论，直接上 LeetCode 或 牛客网 原题。用代码思考。

---

## 1. LeetCode 645. 错误的集合
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/6ed1ce8e55894b8d876e4c55a5ca4a42.png#pic_center)

**主要题意**：在一个本该是 $1$ 到 $n$ 的连续序列里，有一个数重复了，导致另一个数被丢失。怎么快速把这俩揪出来？

**思路**：
+ 1：排序再遍历，太慢了。
+ 2：**拿空间换时间**。开辟一个额外的计数数组（哈希表），原数组里读到几，就在计数数组下标为该值的元素里加一。再循环一遍计数数组，值为 2 的就是重复项，值为 0 的就是丢失项。

> **踩坑批注**：刚开始我直接开了一个 `arr[100000]` 在栈上。虽然能过，但如果题目给的数据量再大点，就Stack Overflow。更优雅的做法是用 `calloc` 动态开辟刚好够用的空间。

**参考代码**：

```c
/**
 * Note: The returned array must be malloced, assume caller calls free().
 */
#include <stdlib.h>

int* findErrorNums(int* nums, int numsSize, int* returnSize) {
    int* ans = (int*)malloc(2 * sizeof(int));
    *returnSize = 2;
    
    // 动态开辟哈希数组，大小为 numsSize + 1，自带初始化为 0
    int* count = (int*)calloc(numsSize + 1, sizeof(int));
    
    for(int i = 0; i < numsSize; i++) {
        count[nums[i]]++; // 核心：以值为下标进行统计
    }
    
    for(int i = 1; i <= numsSize; i++) { // 数据从 1 开始
        if(count[i] == 0) ans[1] = i;    // 没出现过，丢失
        if(count[i] == 2) ans[0] = i;    // 出现两次，重复
    }
    
    free(count); // 别忘了释放内存
    return ans;
}
```

---

## 2. LeetCode 1365. 有多少小于当前数字的数字
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/bdd6290a9baf4eee96b4a39229ac6bf6.png#pic_center)

**主要题意**：对于数组里的每个数，找出比它小的数有多少个。

**思路（频次统计 + 前缀和）**：
+ 1：暴力解法就是两层 `for` 循环嵌套，时间复杂度 $O(N^2)$。当数据量上万时，程序就得跑半天。
+ 2：注意到题目给了一个关键的条件：`0 <= nums[i] <= 100`。数据范围∠小，直接计数排序
+ 2-1. `count[i]`统计每个数字出现的次数。
+ 2-2. 计算**前缀和**。让 `count[i] += count[i-1]`。**小于或等于 $i$ 的数字总共有`count[i]` **。

> **算法思维**：用前缀和处理数组区间问题，能把 $O(N)$ 的区间查询操作直接降维到 $O(1)$。

**参考代码**：

```c
int* smallerNumbersThanCurrent(int* nums, int numsSize, int* returnSize) {
    int* ans = (int*)malloc(numsSize * sizeof(int));
    *returnSize = numsSize;
    
    int count[101] = {0}; // 题目限制0 <= nums[i] <= 100
    
    // 1. 频次统计
    for(int i = 0; i < numsSize; i++) {
        count[nums[i]]++;
    }
    
    // 2. 累加前缀和：此刻 count[i] 代表 <= i 的元素总数
    for(int i = 1; i <= 100; i++) {
        count[i] += count[i-1];
    }
    
    // 3. 将答案保存到ans
    for(int i = 0; i < numsSize; i++) {
        // 如果当前数字是 0，比它小的个数就是 0；否则查表
        ans[i] = (nums[i] == 0 ? 0 : count[nums[i] - 1]);
    }
    
    return ans;
}
```

---

## 3. LeetCode 448. 找到所有数组中消失的数字
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/999cf17203d34bf4891adb1f2cbc3953.png#pic_center)

**主要题意**：这题和第一题极其相似，但进阶要求非常变态：**不使用额外空间，且时间复杂度为 $O(N)$**。（返回的数组不算在内）。

**思路**：
+ 1:直接在**原数组**中操作。
因为数组里的数字都在 $[1, N]$ 范围内，而数组的下标也是 $[0, N-1]$。它们有着天然的一一对应关系（数字 $x$ 对应下标 $x-1$）。遍历数组，每遇到一个数 $x$，把下标为 $x-1$ 的数**变成负数**。遍历完后，谁的位置上还是正数，谁就没出现过。

> **借助库函数（math.h）**：利用 `abs()` 取绝对值，防止已经被打上负号的数字干扰。

**参考代码**：

```c
#include <stdlib.h>
#include <math.h>

int* findDisappearedNumbers(int* nums, int numsSize, int* returnSize) {
    int* res = (int*)malloc(numsSize * sizeof(int));
    int j = 0;
    
    // 1. 第一层遍历：进行负号标记
    for(int i = 0; i < numsSize; i++) {
        int index = abs(nums[i]) - 1; // 拿到数字对应的真实下标
        if(nums[index] > 0) {
            nums[index] = -nums[index]; // 标记为负数，证明 abs(nums[i]) 来过
        }
    }
    
    // 2. 第二层遍历：查漏补缺
    for(int i = 0; i < numsSize; i++) {
        if(nums[i] > 0) { 
            // 如果还是正数，说明 i+1 这个数字根本没出现过
            res[j++] = i + 1;
        }
    }
    
    *returnSize = j;
    return res;
}
```

---

## 4. LeetCode 28. 找出字符串中第一个匹配项的下标
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/6bb2ad1c274640d8b4f33441f0dd6a89.png#pic_center)

**主要题意**：在长字符串（主串）中寻找短字符串（模式串）。
### 优解一：KMP 算法

KMP 算法的核心：**主串指针绝不回退**。
它通过预处理模式串，生成一个 `next` 数组（最长公共前后缀表）。当发生不匹配时，它能告诉你模式串该往前滑动多少，而不是像傻子一样每次只挪一位。

```c
// KMP 算法实现
int strStr_KMP(char* haystack, char* needle) {
    int n = strlen(haystack), m = strlen(needle);
    if (m == 0) return 0;
    
    int* next = (int*)malloc(m * sizeof(int));
    next[0] = 0;
    
    // 1. 构建 next 数组
    for (int i = 1, j = 0; i < m; i++) {
        while (j > 0 && needle[i] != needle[j]) {
            j = next[j - 1]; // 不匹配，回退 j
        }
        if (needle[i] == needle[j]) j++;
        next[i] = j;
    }
    
    // 2. 正式匹配
    for (int i = 0, j = 0; i < n; i++) {
        while (j > 0 && haystack[i] != needle[j]) {
            j = next[j - 1]; // 核心：主串 i 不回退，只回退模式串 j
        }
        if (haystack[i] == needle[j]) j++;
        if (j == m) {
            free(next);
            return i - m + 1; // 匹配成功
        }
    }
    free(next);
    return -1;
}
```

### 优解二：Sunday 算法

KMP 虽强，但太难记了。在实际应用中，**Sunday 算法**跑得更快，而且逻辑更简单。
**它的核心是看主串匹配窗口的“下一个字符”**：
如果在模式串里根本没这个字符，直接把模式串整个滑过这个字符！

```c
// Sunday 算法实现
int strStr(char* haystack, char* needle) {
    int n = strlen(haystack), m = strlen(needle);
    if (m == 0) return 0;
    
    // 1. 构建偏移表：记录字符最右出现位置距离尾部的距离
    int shift[256];
    for (int i = 0; i < 256; i++) shift[i] = m + 1; // 默认偏移整个模式串长度+1
    for (int i = 0; i < m; i++) shift[(unsigned char)needle[i]] = m - i;
    
    int i = 0;
    // 2. 开始匹配
    while (i <= n - m) {
        int j = 0;
        while (j < m && haystack[i + j] == needle[j]) j++; // 逐个比对
        
        if (j == m) return i; // 完全匹配
        if (i + m >= n) return -1; // 边界判断
        
        // 核心：根据主串参加匹配的【下一个字符】的偏移量，决定向右跳多远
        i += shift[(unsigned char)haystack[i + m]];
    }
    return -1;
}
```
> **建议**：如果要求 $O(M+N)$ 的复杂度， KMP；否则，Sunday 算法的代码量和执行效率更优。

---
