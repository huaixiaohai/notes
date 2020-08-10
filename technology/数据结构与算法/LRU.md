Least Recently Used,最近最久未使用法，它是按照一个非常著名的计算机操作系统基础理论得来的：**最近使用的页面数据会在未来一段时期内仍然被使用,已经很久没有使用的页面很有可能在未来较长的一段时间内仍然不会被使用**。

基于这个思想, 会存在一种缓存淘汰机制，每次从内存中找到最久未使用的数据然后置换出来，从而存入新的数据！它的主要衡量指标是使用的时间，附加指标是使用的次数。

**在计算机中大量使用了这个机制**，它的合理性在于优先筛选热点数据，所谓热点数据，就是最近最多使用的数据！因为，利用LRU我们可以解决很多实际开发中的问题，并且很符合业务场景。



代码实现：

​		这里筛选的指标只是简单的关注最近使用的时间，不关注次数。

​		实现数据结构：双向链表+hashtable，

​				hashtable主要是为了在O(1)的时间复杂度查找节点，主要是辅助双向链表操作

​				双向链表主要先进先出(FIFO)，越靠近头部数据越容易被使用，优先移除链尾数据。

​		注意：

​				新增的节点，添加到链表的头部

​				新增超出链表长度，优先移除链表尾部数据

​				每次查找数据，需要把改数据添加到链表头部					

golang 语言实现							

```go
type Node struct {
	Data int
	Key  int
	Next *Node
	Pre  *Node
}

type LRUCache struct {
	Dict      map[int]*Node
	DLinkList *Node
	Capacity  int
}

func Constructor(capacity int) LRUCache {
	return LRUCache{
		Capacity:  capacity,
		Dict:      make(map[int]*Node, 0),
		DLinkList: nil,
	}
}

func (this *LRUCache) Get(key int) int {
	node, has := this.Dict[key]
	if !has {
		return -1
	}
	if node != this.DLinkList {
		node.Pre.Next = node.Next
		node.Next.Pre = node.Pre
		node.Next = this.DLinkList
		node.Pre = this.DLinkList.Pre
		this.DLinkList.Pre.Next = node
		this.DLinkList.Pre = node
		this.DLinkList = node
	}
	return node.Data
}

func (this *LRUCache) Put(key int, value int) {
	node, has := this.Dict[key]
	if has {
		if node != this.DLinkList {
			node.Pre.Next = node.Next
			node.Next.Pre = node.Pre
			node.Next = this.DLinkList
			node.Pre = this.DLinkList.Pre
			this.DLinkList.Pre.Next = node
			this.DLinkList.Pre = node
			this.DLinkList = node
		}
		node.Data = value
		return
	}

	if len(this.Dict) >= this.Capacity {
		delete(this.Dict, this.DLinkList.Pre.Key)
		this.DLinkList = this.DLinkList.Pre
		this.DLinkList.Key = key
		this.DLinkList.Data = value
		this.Dict[this.DLinkList.Key] = this.DLinkList
		return
	}

	node = &Node{
		Data: value,
		Key:  key,
		Next: nil,
		Pre:  nil,
	}
	this.Dict[key] = node
	if this.DLinkList == nil {
		this.DLinkList = node
		this.DLinkList.Pre = this.DLinkList
		this.DLinkList.Next = this.DLinkList
	} else {
		node.Pre = this.DLinkList.Pre
		node.Next = this.DLinkList
		this.DLinkList.Pre.Next = node
		this.DLinkList.Pre = node
		this.DLinkList = node
	}
}
```


