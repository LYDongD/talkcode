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
    
