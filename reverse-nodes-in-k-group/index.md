# LeetCode面试高频题系列二：K个一组翻转链表


<!--more-->

题目链接：[https://leetcode-cn.com/problems/reverse-nodes-in-k-group/](https://leetcode-cn.com/problems/reverse-nodes-in-k-group/)

> 写在前面：几乎所有链表题都有**迭代**和**递归**两种方法，并且两种方法都应该掌握，因为有时候面试官会要求使用某种特定的方法。

## 迭代方法
~~~go
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func reverseKGroup(head *ListNode, k int) *ListNode {
    // 满足以下任何一个条件，直接返回 head
    if head == nil || head.Next == nil || k == 1 {
        return head
    }

    // 每次翻转需要保留上一次翻转的最后一个节点，为了让第一次翻转和后续的翻转操作统一这里我们使用虚拟节点
    dummy := &ListNode{Next: head}
    pre, left, right := dummy, head, head
    for {
        for i := 0; i < k; i++ {
            // 不足K个直接返回
            if right == nil {
                return dummy.Next
            }
            right = right.Next
        }
        // 上一次翻转的最后一个节点应该指向当前翻转后的头结点
        pre.Next = reverseBetween(left, right)
        // pre 变为当前翻转后的最后一个节点，即 left
        pre = left
        // left 变为下一次翻转的头结点，即 right
        left = right
    }
    
}

// Reverse [head, tail)
// 下方的函数类似单链表反转，不懂的话可以参考我的另一篇文章：https://www.peileiscott.top/reverse-linked-list/
func reverseBetween(head, tail *ListNode) *ListNode {
    if head == tail || head.Next == tail {
        return head
    }
    
    prev, curr := tail, head
    for curr != tail {
        next := curr.Next
        curr.Next = prev
        prev = curr
        curr = next
    }
    return prev
}
~~~

## 递归方法
~~~go
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func reverseKGroup(head *ListNode, k int) *ListNode {
    if head == nil || head.Next == nil || k == 1 {
        return head
    }
    
    p := head
    for i := 0; i < k; i++ {
        // 不足K个直接返回
        if p == nil {
            return head
        }
        p = p.Next
    }
    
    // 翻转 [head, p) 之间的K个节点
    newHead := reverse(head, p)
    // 翻转后前K个节点的最后一个节点为 head，指向后续翻转返回的头节点
    head.Next = reverseKGroup(p, k)
    // 返回新的头节点
    return newHead
}

// Reverse [head, tail)
// 下方的函数类似单链表反转，不懂的话可以参考我的另一篇文章：https://www.peileiscott.top/reverse-linked-list/
func reverseBetween(head, tail *ListNode) *ListNode {
    if head == tail || head.Next == tail {
        return head
    }
    
    prev, curr := tail, head
    for curr != tail {
        next := curr.Next
        curr.Next = prev
        prev = curr
        curr = next
    }
    return prev
}
~~~

## 题目拓展

### 不足K个也翻转
> 个人认为这个拓展反而让题目变简单了，在这里只给出迭代方法，递归方法可以类似得出。
~~~go
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */

// 不足K个也翻转
func reverseKGroup(head *ListNode, k int) *ListNode {
    // 满足以下任何一个条件，直接返回 head
    if head == nil || head.Next == nil || k == 1 {
        return head
    }

    // 每次翻转需要保留上一次翻转的最后一个节点，为了让第一次翻转和后续的翻转操作统一这里我们使用虚拟节点
    dummy := &ListNode{Next: head}
    pre, left, right := dummy, head, head
    for {
        for i := 0; i < k; i++ {
            // 不足K个直接开始翻转并返回结果
            if right == nil {
                pre.Next = reverseBetween(left, right)
                return dummy.Next
            }
            right = right.Next
        }
        // 上一次翻转的最后一个节点应该指向当前翻转后的头结点
        pre.Next = reverseBetween(left, right)
        // pre 变为当前翻转后的最后一个节点，即 left
        pre = left
        // left 变为下一次翻转的头结点，即 right
        left = right
    }
    
}

// Reverse [head, tail)
// 下方的函数类似单链表反转，不懂的话可以参考我的另一篇文章：https://www.peileiscott.top/reverse-linked-list/
func reverseBetween(head, tail *ListNode) *ListNode {
    if head == tail || head.Next == tail {
        return head
    }
    
    prev, curr := tail, head
    for curr != tail {
        next := curr.Next
        curr.Next = prev
        prev = curr
        curr = next
    }
    return prev
}
~~~

### 从尾部开始计数
> 思路：先计算链表的长度，从而计算出从哪个节点开始翻转。也可以用栈做，但是那样空间复杂度就为O(n)了，这里就不给出栈的做法了，有兴趣的可以自己试试。
~~~go
// 从尾部开始计数
func reverseKGroup(head *ListNode, k int) *ListNode {
	// 满足以下任何一个条件，直接返回 head
	if head == nil || head.Next == nil || k == 1 {
		return head
	}

	// n 用来记录链表的长度
	p, n := head, 0
	for p != nil {
		n++
		p = p.Next
	}

	// 每次翻转需要保留上一次翻转的最后一个节点，为了让第一次翻转和后续的翻转操作统一这里我们使用虚拟节点
	dummy := &ListNode{Next: head}
	pre, left, right := dummy, head, head

    // 先让 pre, left, right 定位到第一次翻转的位置
	for i := 0; i < n%k; i++ {
		pre = pre.Next
		left = left.Next
		right = right.Next
	}

	for {
		for i := 0; i < k; i++ {
			// 最后一次翻转时 right 会等于 nil，进入 if 内部进行最后一次翻转，并返回结果
			if right == nil {
				pre.Next = reverseBetween(left, right)
				return dummy.Next
			}
			right = right.Next
		}
		// 上一次翻转的最后一个节点应该指向当前翻转后的头结点
		pre.Next = reverseBetween(left, right)
		// pre 变为当前翻转后的最后一个节点，即 left
		pre = left
		// left 变为下一次翻转的头结点，即 right
		left = right
	}

}

// Reverse [head, tail)
// 下方的函数类似单链表反转，不懂的话可以参考我的另一篇文章：https://www.peileiscott.top/reverse-linked-list/
func reverseBetween(head, tail *ListNode) *ListNode {
	if head == tail || head.Next == tail {
		return head
	}

	prev, curr := tail, head
	for curr != tail {
		next := curr.Next
		curr.Next = prev
		prev = curr
		curr = next
	}
	return prev
}
~~~
