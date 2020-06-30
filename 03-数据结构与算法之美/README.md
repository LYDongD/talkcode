## 数据结构与算法之美

#### [05 | 数组：为什么很多编程语言中数组都从0开始编号？](https://time.geekbang.org/column/article/40961)

> 为什么说数组随机访问的时间开销是O(1)

* 数组空间是连续的，只需要确定首元素的地址，就能O1的算出其他元素的地址
    * 一维数组：address = base_address + index * type_size
    * 二维数组(m,n) ：address = base_address + (i * n + j) * type_size
    * 以上公式的前提是数组索引从0开始编号
        * 如果从1开始，公式变为address = base_address + （index-1） * type_size
    	* 多了一次减法操作，相当于多了一条指令
    	* 避免性能开销，使用0开始编号
* 数组如何扩容？
    * 创建新的容量更大的数组（分配新的连续的内存空间）
    * 复制旧数组的元素到新数组
    * 扩容最差情况下需要On的时间开销
    * hashmap底层也是数组，区别是数组元素不是一一对应复制的，而是会通过rehash重新计算索引
 

#### [06 | 链表（上）：如何实现LRU缓存淘汰算法?](https://time.geekbang.org/column/article/41013)

* 链表有哪些分类？
    * 单向链表
	* next
    * 双向链表
	* next/prev
	    * 多了prev，删除特定节点的时候时间开销为O(1)
	    * 另外可以实现从后向前遍历，某些时候从后向前效率更高
    * 循环链表（单向/双向）
	* tail.next = head
	* head.prev = tail

* 如何用链表实现LRU缓存
    * 读 -> 删除读的节点，插入队尾
    * 插入 -> 缓存是否满
	* 满 -> 删除队头并插入队尾
	* 未满 -> 插入队尾
    * LinkedHashMap是如何实现LRU缓存的？
	* todo
