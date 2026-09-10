@[TOC](链表双指针与二级指针)

# 玩转离散内存与映射

今天我们将战场转移到**链表**和**哈希映射**上。

链表的痛点在于“单向不回头”和“无法随机访问”。实际上，搭配**指针**（快慢指针、二级指针），链表的很多操作远比数组优雅。

直接上题，看代码。

---

## 1. LeetCode 876. 链表的中间结点
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/f26d1bf26cec4e138417cb977d3800d8.png#pic_center)

**题意分析**：单链表不知道总长度，想找中间节点，通常得先遍历一遍求长度，然后再遍历一半找节点。跑两趟，太笨重了。

**思路**：
**快慢指针（双指针法）**。
一条跑道，兔子和乌龟同时起跑。兔子每次跑两步（`fast = fast->next->next`），乌龟每次跑一步（`slow = slow->next`）。
当兔子冲过终点线时，乌龟绝对刚好踩在半程的线上。


> **踩坑批注**：注意 `while` 循环的边界条件 `fast && fast->next`。因为 `fast` 一次要跳两步，必须保证它当前节点和下一个节点都不是 `NULL`，否则直接报段错误（解引用空指针）。

**参考代码**：

```c
/**
 * Definition for singly-linked list.
 * struct ListNode {
 * int val;
 * struct ListNode *next;
 * };
 */
struct ListNode* middleNode(struct ListNode* head) {
    if(!head || !head->next) return head; // 只有0个或1个节点，直接返回
    
    struct ListNode* fast = head;
    struct ListNode* slow = head;
    
    // fast->next只要不是NULL，循环就继续
    while(fast && fast->next) {
        fast = fast->next->next; // 兔子走两步
        slow = slow->next;       // 乌龟走一步
    }
    
    return slow; // 兔子到终点，乌龟刚好在中间
}
```

---

## 2. LeetCode 383. 赎金信
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/d5132735e5ec4dc0956ded374d397d37.png#pic_center)

**题意分析**：判断字符串 A 能不能由字符串 B 里的字符拼出来，且 B 里的字符只能用一次。

**思路**：
这题是典型的字符统计问题。因为题目明确提示“由小写英文字母组成”，所以字母种类最多也就 26 个（或者说 ASCII 字符最多 256 个）。
可以直接开一个固定大小的数组当哈希表。
1. 遍历 `magazine`（杂志），把拥有的字符作为数组下标，对应位置的频次加一。
2. 遍历 `ransomNote`（赎金信），把需要的字符作为数组下标，对应位置的频次减一。
3. 如果减完之后频次小于 0，说明“杂志里的库存”不够用，直接判负！

> **故事拓展**：以前绑匪为了不暴露字迹，拿一页杂志，在上面圈字母，组成一句话，用来勒索。
> 例句： The kidnappers circled letters in the magazine to form a ransom
> note in order not to reveal their handwriting.
> 绑匪为了不暴露字迹，在杂志上圈选字母组成赎金信。

**参考代码**：

```c
#include <stdbool.h>

bool canConstruct(char* ransomNote, char* magazine) {
    //  ASCII 码映射，255 容量足以覆盖所有基础字符
    // 如果想更省内存，可以开 int arr1[26] = {0}; 然后下标用 [magazine[i] - 'a']
    int arr1[255] = {0}; 
    
    // 1. 录入库存
    for(int i = 0; magazine[i] != '\0'; i++) {
        arr1[magazine[i]]++;
    }
    
    // 2. 消耗库存并实时校验
    for(int i = 0; ransomNote[i] != '\0'; i++) {
        // 先减 1，如果减完发现变成负数了，说明不够用
        if((--arr1[ransomNote[i]]) < 0) {
            return false;
        }
    }
    
    return true;
}
```

---

## 3. LeetCode 2. 两数相加
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/40f1a9b7e70e481b9a70e00639738186.png#pic_center)

**题意分析**：两个链表逆序存储数字，要求按位相加并处理进位。难点不在于数学逻辑，而在于**如何优雅地处理链表的新增节点和头节点的特殊情况**。

> 通常做法是建一个哑节点（Dummy Node），这样可以避免判断 `if(head == NULL)` 的繁琐逻辑。
> 但是，采用**二级指针（Pointer to Pointer）**法，极具艺术感！Linux 之父 Linus Torvalds 曾把这种写法称为 "Good Taste"。


**思路**：
`struct ListNode** link` 这个二级指针，存的永远是**“下一个需要被赋值的指针的地址”**。
* 初始时，它指向 `head` 的地址。我们给 `*link` 分配内存，其实就是把新节点挂在了 `head` 上。
* 随后，执行 `link = &((*link)->next);`。这时候 `link` 指向了刚刚建好的那个新节点的 `next` 指针的地址。
* 下一轮循环时，再去给 `*link` 分配内存，就顺理成章地连到了整个链表的尾巴上。全程不需要 Dummy Node，也不需要单独记录 tail 指针！

> 代码解析：此处提供3种解法

**参考代码**：

```c
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     struct ListNode *next;
 * };
 */
// struct ListNode* addTwoNumbers(struct ListNode* l1, struct ListNode* l2) {
//     // struct ListNode dummy;
//     // dummy.next=NULL;
//     // struct ListNode* curr=&dummy;
//     // int carry=0;
//     // while(l1!=NULL || l2!=NULL || carry!=0)
//     // {
//     //     int sum=carry;
//     //     if(l1!=NULL)
//     //     {
//     //         sum+=l1->val;
//     //         l1=l1->next;
//     //     }
//     //     if(l2!=NULL)
//     //     {
//     //         sum+=l2->val;
//     //         l2=l2->next;
//     //     }
//     //     carry=sum/10;
//     //     struct ListNode* newNote=(struct ListNode*)malloc(sizeof(struct ListNode));
//     //     newNote->next=NULL;
//     //     newNote->val=sum%10;
//     //     curr->next=newNote;
//     //     curr=curr->next;
//     // }
//二
//     struct ListNode dummy;
//     dummy.next=l1;
//     struct ListNode* prev=&dummy;
//     int carry=0;
//     int sum=0;
//     while(l1!=NULL && l2!=NULL)
//     {
//         sum=l1->val+l2->val+carry;
//         l1->val=sum%10;
//         carry=sum/10;
//         prev=l1;
//         l1=l1->next;
//         l2=l2->next;
//     }
//     if(l2!=NULL)
//     {
//         prev->next=l2;
//         l1=l2;
//     }
//     while(l1!=NULL && carry>0)
//     {
//         sum=l1->val+carry;
//         l1->val=sum%10;
//         carry=sum/10;
//         prev=l1;
//         l1=l1->next;
//     }
//     if(carry>0)
//     {
//         struct ListNode* newNode=(struct ListNode*)malloc(sizeof(struct ListNode));
//         newNode->val=carry;
//         newNode->next=NULL;
//         prev->next=newNode;
//     }
//     return dummy.next;
// }
//三
struct ListNode* addTwoNumbers(struct ListNode* l1, struct ListNode* l2) {
    struct ListNode* head = NULL;
    // link 指向“待连接的那个指针的地址”
    struct ListNode** link = &head; 
    int carry = 0;

    // 只要 l1 没走完，或者 l2 没走完，或者最后还有进位，就继续加
    while (l1 || l2 || carry) {
        int sum = carry;
        if (l1) { sum += l1->val; l1 = l1->next; }
        if (l2) { sum += l2->val; l2 = l2->next; }

        carry = sum / 10;

        // 直接在二级指针指向的地方原地分配新节点
        *link = (struct ListNode*)malloc(sizeof(struct ListNode));
        (*link)->val = sum % 10;
        (*link)->next = NULL;

        // 灵魂一步：把 link 挪到新节点的 next 属性的地址上，准备迎接下一个节点
        link = &((*link)->next); 
    }

    return head;
}
```

---

继续刷题，下期见！
