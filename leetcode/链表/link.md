// 定义单向链表
type ListNode struct {
    Val  int
    Next *ListNode
}

# 环形链表

题号：141

给你一个链表的头节点 head ，判断链表中是否有环。
如果链表中有某个节点，可以通过连续跟踪 next 指针再次到达，则链表中存在环。
为了表示给定链表中的环，评测系统内部使用整数 pos 来表示链表尾连接到链表中的位置（索引从 0 开始）。
如果 pos 是 -1，则在该链表中没有环。注意：pos 不作为参数进行传递，仅仅是为了标识链表的实际情况。
如果链表中存在环，则返回 true 。 否则，返回 false

```go
type ListNode struct {
 Val int
 Next *ListNode
}

func hasCycle(head *ListNode) bool {
    // 创建一个map存储遍历过的节点
    m := make(map[*ListNode]bool)
    for head != nil {
        // 如果节点已经遍历过，说明存在环，则返回true
        if _, ok := m[head]; ok {
            return true
        } else {
            // 如果没有遍历过，则放入map中，并遍历下一个节点
            m[head] = true
            head = head.Next
        }
    }
    // 全部遍历完，没有找到环，则返回false
    return false
}
```

# 142. 环形链表 II

题号： 142

给定一个链表，返回链表开始入环的第一个节点。 如果链表无环，则返回 null。
如果链表中有某个节点，可以通过连续跟踪 next 指针再次到达，则链表中存在环。
为了表示给定链表中的环，评测系统内部使用整数 pos 来表示链表尾连接到链表中的位置（索引从 0 开始）。
如果 pos 是 -1，则在该链表中没有环。注意：pos 不作为参数进行传递，仅仅是为了标识链表的实际情况。
不允许修改 链表。

```go
type ListNode struct {
 Val int
 Next *ListNode
}

func detectCycle(head *ListNode) *ListNode {
    m := make(map[*ListNode]bool)
    for head != nil {
        if _, ok := m[head]; ok {
            // 返回入环的第一个节点
            return head
        } else {
            m[head] = true
            head = head.Next
        }
    }
    return nil
}
```
