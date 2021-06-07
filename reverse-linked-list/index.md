# LeetCode面试高频题系列一：单链表翻转


<!--more-->

题目链接：[https://leetcode-cn.com/problems/reverse-linked-list/](https://leetcode-cn.com/problems/reverse-linked-list/)
> 写在前面：几乎所有链表题都有**迭代**和**递归**两种方法，并且两种方法都应该掌握，因为有时候面试官会要求使用某种特定的方法。

## 迭代方法
由于单链表节点中只保留着下一个节点的信息，上一个节点的信息我们是不知道的，而翻转过程中我们需要将当前节点的`next`指向上一个节点，因此我们在遍历的过程中需要额外用一个指针告诉我们上一个节点；另外当我们修改了当前节点的`next`值后，我们将无法通过`next`来获取下一个节点，因此在修改`next`之前我们要用一个临时变量获取下一个节点。

> 通过上面分析，我们在代码实现中只需要注意两点即可：
>
> 1. 遍历过程中用一个指针`pre`指向上一个节点
> 2. 修改当前节点的`next`之前用一个临时变量获取下一个节点

经过上面的分析过后，不难写出下面代码：
~~~go
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func reverseList(head *ListNode) *ListNode {
    // 当链表为空或链表只有一个节点时，直接返回
    if head == nil || head.Next == nil {
        return head
    }

    // pre 指向上一个节点，由于第一个节点的 next 将指向 nil，因此为了方便我们可以将 pre 初始化为 nil
    pre, curr := (*ListNode)(nil), head

    for curr != nil {
        // 临时变量 next 获取当前节点 curr 的下一个节点
        next := curr.Next
        // 修改当前节点 curr 的 Next 值完成翻转
        curr.Next = pre
        pre = curr
        curr = next
    }

    // 注意此时 curr 已经变成了 nil, pre 指向了原链表的最后一个节点(新链表的头结点)
    return pre
}
~~~

## 递归方法

![reverse-linked-list](https://cdn.jsdelivr.net/gh/PeiLeiScott/image-hosting@master/reverse-linked-list.3q90u0piw0k0.png)

~~~go
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func reverseList(head *ListNode) *ListNode {
    // 递归终止条件
    if head == nil || head.Next == nil {
        return head
    }

    // 翻转以 head 的下一个节点为首的链表，并获取翻转后的链表的头结点
    newHead := reverseList(head.Next)
    // 将 head 的下一个节点指向 head (将上图节点2指向节点1)
    head.Next.Next = head
    // 将 head 指向 nil (将上图节点1指向 nil)
    head.Next = nil 
    // 返回新的头结点 newHead
    return newHead
}
~~~

