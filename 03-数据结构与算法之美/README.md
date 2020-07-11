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

#### [07 | 链表（下）：如何轻松写出正确的链表代码？](https://time.geekbang.org/column/article/41149)

* 如何逻辑清晰的实现链表相关的算法？
    * 理解指针的含义
	    * next -> 后继节点
	    * prev -> 前驱节点
    * 警惕指针丢失
	* 注意指针更新的顺序
	    * 例如更新了next后再使用next已经不是原来的next
	* 释放内存
	    * 设置无效指针 = null 可以提醒GC将其删除
    * 哨兵节点
	    * 处理链表边界条件，使之和非边界节点的处理逻一致
	    * 避免用if-else特殊处理边界情况
    * 注意边界条件
	    * 事实上在任何算法的逻辑里面都要考虑边界条件
    * 画图
	    * 非常有效，无论是算法还是某个知识体系，画图并能够在脑海中形成清晰的图像，利于记忆和理顺逻辑
	    * 写算法时尽量画图，写伪代码
        * 多写多练
	* 留下5个todo
	    * 单链表反转
	    * 链表中环的检测
	    * 两个有序的链表合并
	    * 删除链表倒数第 n 个结点
	    * 求链表的中间结点

#### [08 | 栈：如何实现浏览器的前进和后退功能？](https://time.geekbang.org/column/article/41222)

> 栈在jvm内存管理中有哪些应用？

* 方法调用栈
    * 每个线程维护一个call stack
        * 调用方法 -> 入栈
        * 方法返回 -> 出栈
    * 栈帧的组成
        * 变量表
        * 方法返回地址
        * 操作数栈
            * 保存操作数 -> load/store指令用于在变量表和操作数栈之间移动指令

> 栈与递归的关系

* 递归调用实际上使用了系统分配的隐式方法调用栈
    * 如果递归没有恰当结束或递归深度过大，可能引发stack over flow

> 用栈解决的经典问题

* 浏览器切换
    * 双栈
* 四则运算
    * 双栈
* 括号匹配
    * 单栈

#### [09 | 队列：队列在线程池等有限资源池中的应用](https://time.geekbang.org/column/article/41330)

> 循环队列怎么实现？它解决了什么问题？

循环队列本质上也是通过数组实现，和顺序队列相比，tail指针可以移动到head指针的左边，形成头尾循环；
相比顺序队列，如果要求tail在head的右边，当tail达到数组末尾时，需要搬移数据，性能有所损耗。总结来说，
循环队列避免了顺序队列在入队出队过程中可能发生的数据迁移操作。

实现队列有两个关键：

1. 入队和出队时怎么判满和判空，例如顺序队列和循环队列的判断条件不一样
2. 指针的移动，例如循环队列，顺序队列和链式队列指针的移动方式不同
    * 对于数组，tail总是指向下一个空元素；
    * 循环数组需要空闲一个位置用于区分空和满的判别算法
    * 链表的tail执行尾部元素

> 队列使用场景

* 阻塞队列
    * 用于实现任务的生产消费者模式
        * 协调生产和消费线程的速率
        * 速率不对等将导致队列为空或爆满，通过阻塞相应线程来平衡速率
    * 应用
        * ArrayBlockingQueue
        * LinkedBlockingQueue

* 并发队列
    * 生产线程和消费线程可同时操作队列并保证线程安全
        * 通过锁或cas机制保证指针等操作的原子性
            * disruptor使用cas + 循环队列
            * ArrayBlockingQueue使用一把锁管理入队出队（todo)
            * LinkedBlocking分别用2把锁管理入队出队(todo)
    * 如何提高多线程消费队列的性能？
        * 可以使用多个队列，每个队列单线程消费，则无需加锁 
            * 例如kafka，分布式队列分区，一个分区是一个队列，只能被一个线程消费
        * 使用CAS
            * 例如disruptor，采用乐观锁避免线程阻塞

#### [10 | 递归：如何用三行代码找到“最终推荐人”？](https://time.geekbang.org/column/article/41440)

* 什么问题可以用递归解决？
    * 一个问题可以分解为几个子问题的解
    * 子问题和当前问题的求解方式一致
    * 存在终止条件

* 如何找出递归的递推公式？
    * 类比证明中的数学归纳法
        * 计算初始条件值：f(1)
        * 在f(n-1)成立的情况下证明f(n)成立
            * 在f(n-1)成立的情况下，如何求解f(n)

* 递归存在的问题
    * 递归深度过大导致栈溢出
        * 限制栈深度
    * 重复计算
        * 使用缓存解决
    * 如何处理无限递归的问题
        * 例如 A -> B -> C -> A
            * 缓存已经处理过的节点，例如A节点
            * 处理C时，优先从缓存取，没有才创建A
            * 以上方法可以解决spring依赖注入时循环依赖的问题

#### [11 | 排序（上）：为什么插入排序比冒泡排序更受欢迎？](https://time.geekbang.org/column/article/41802)

> 如何实现一个冒泡排序

优化点：

1. 逆序度为0时提前退出：每一轮记录是否发生过交换
2. 交换操作：用位异或运算进行交换

```
func bubbleSort(nums []int) {
    if len(nums) <= 1 {
        return
    }

    n := len(nums)
    for i := 0; i < n; i++ {
        hasSwap := false
        for j := 0; j < n-i-1; j++ {
            if nums[j] > nums[j+1] {
                swap(nums, j, j+1)
                hasSwap = true
            }
        }

        if !hasSwap {
            break
        }
    }
}

func swap(nums []int, a, b int) {
    tmp := nums[a] ^ nums[b]
    nums[a] = tmp ^ nums[a]
    nums[b] = tmp ^ nums[b]
}

```

> 如何实现插入排序？

参考jdk的排序算法qSort, 数据规模较小时采用插入排序，查找插入位置时不时从后往前，而是通过二分法查找

```
func insertSort(nums []int) {
    if len(nums) <= 1 {
        return
    }

    n := len(nums)
    for i := 1; i < n; i++ {
        j := i - 1
        num := nums[i]
        for ; j >= 0; j-- {
            if nums[j] > num {
                nums[j+1] = nums[j]
            } else {
                break
            }
        }

        nums[j+1] = num
    }
}

```

> 如何评价排序算法的好坏？

3个指标：

* 时间复杂度（最好，最差和平均）
* 空间复杂度
* 稳定性 -> 排序过程中是否会改变相同元素的相对位置
    * 稳定性用于在多因素排序的场景中，例如先根据A排序，相同的A里面根据B排序
        * 先根据B排序，再根据A排序，因为稳定性，相同的A的B是有序的，不会发生变化


#### [12 | 排序（下）：如何用快排思想在O(n)内查找第K大元素？](https://time.geekbang.org/column/article/41913)

> 如何在无序数组中查找第K大的元素？

* 分区 + 递归
    * 分区后，判断第K大元素在哪个区间，对该区间递归查找

```
func findK(nums []int, k int) int {
    if len(nums) < k {
        panic("k must not excess nums' bound")
    }

    return findKRecursively(k, nums, 0, len(nums)-1)
}

func findKRecursively(k int, nums []int, start, end int) int {
    if start == end {
        return nums[start]
    }

    pivot := nums[end]
    j := start
    for i := start; i < end; i++ {
        if nums[i] < pivot {
            swap(nums, i, j)
            j++
        }
    }

    swap(nums, j, end)

    //find next range
    if j+1 == k {
        return nums[j]
    } else if j < k {
        return findKRecursively(k, nums, j+1, end)
    } else {
        return findKRecursively(k, nums, start, j-1)
    }

}

func swap(nums []int, a, b int) {
    nums[a], nums[b] = nums[b], nums[a]
}

```

> 快排和归并有什么区别？

* 快排 = 分区 + 递归 
    * 先分区，完成后进行递归  
    * 不是稳定排序，比较交换时可能会将相同元素中前面的交换到后面
* 归并 = 递归 + 合并 
    * 先递归，返回时合并
    * 需要借助临时数组，不是原地排序
    * 可实现文件大数据排序
        * 分割：从文件读指定大小的数据，排序，写入子文件
        * 归并：假设有10个已经排好序的子文件，准备10个并行的io流(指针），基于文件指针，每次读10个子文件各一条数据，选择最小的一个缓存，并更新该文件指针
        * 合并：缓存达到一定规模时写入文件，该文件是全局有序的

#### [13 | 线性排序：如何根据年龄给100万用户数据排序？](https://time.geekbang.org/column/article/42038)

> 线性排序算法有哪些，适用于什么场景

* 桶排序 -> 分组
    * 适用于数据范围不大的情况，按范围分组
    * 用链表实现组，可动态增长
    * 分组后，针对组内元素进行快排
    * 合并多个分组到一个组

* 计数排序 -> 分组 + 计数
    * 适用于数据范围不大的情况，每个分组只有一个值
    * 对数据进行分组，分配到一个数组，索引为数据值，存储该值得数量
    * 从左往右累计，计算每个元素的排位
    * 从后往前遍历原数组进行排位

* 基数排序 -> 分解 + 桶排序/计数排序
    * 计数元素可以被分解成位，元素的顺序由位顺序决定
    * 位排序：分桶或计数排序
    * 综合多个位的排序


#### [14 | 排序优化：如何实现一个通用的、高性能的排序函数？](https://time.geekbang.org/column/article/42359)

> 如何优化快排

快速排序最坏情况下时间效率退化成O(n2)，当分区点无法均匀的进行分区时，例如数据本身是顺序或逆序，但是选择的分区点在末尾时。

因此优化快排，就是要避免总是选择到最差的分区点，可以采用以下策略:

* 三数取中法
    * 收尾中各取一个点，取中间值作为分区点（类似于hashmap算key索引时，将高位和低位进行异或）
	* 目的是综合考虑全局的情况，避免陷入极端情况
* 随机法
    * 随机选择一个分区点，保证分区点不会总是最差的
	* 本质上是利用概率解决极端问题


####[16 | 二分查找（下）：如何快速定位IP对应的省份地址？](https://time.geekbang.org/column/article/42733)

> 二分法变式

* 存在相同元素时，如何查找第一个和最后一个
* 不要求相等，如何查找第一个或最后一个大于等于或小于等于target的元素
* 如何在循环数组中查找目标元素

> 有序数组中查找第一个和目标值相等的元素

```
//find first ele equal 2 target
func search(nums []int, target int) int {
    if len(nums) == 0 {
        return -1
    }

    start, end := 0, len(nums)-1
    for start <= end {
        mid := start + (end-start)>>1
        if nums[mid] > target {
            end = mid - 1
        } else if nums[mid] < target {
            start = mid + 1
        } else {
            //key: found equal and check if it's the first one 
            if mid == 0 || nums[mid-1] != target {
                return mid
            }

            end = mid - 1
        }
    }

    return -1
}

```

> 有序数组中查找最后一个小于或等于目标值的元素

```
//find last ele less than target
func search2(nums []int, target int) int {
    if len(nums) == 0 {
        return -1
    }

    start, end := 0, len(nums)-1
    for start <= end {
        mid := start + (end-start)>>1
        if nums[mid] > target {
            mid = end - 1
        } else {
            //check if it's the last one less than target
            if mid == len(nums)-1 || nums[mid+1] > target {
                return mid
            }

            start = mid + 1
        }
    }

    return -1
}

```

> 在一个循环数组中查找目标元素

循环数组可以分成两个区间: 升序的大区间和升序的小区间，并且所有元素满足 大区间 > 小区间

和普通数组不同的是，当我们发现 target < mid 时，target有可能在右边区间：

1. mid在大区间 -> nums[mid] > tail
2. target在小区间 -> target <= tail

同理，当target > mind时，target有可能在左边区间：

1. mid在小区间 -> nums[mid] < head
2. target在大区间 -> target >= head

综上，在普通数组二分法的基础上增加大小区间的判断。二分法本质上还是一个怎么选择下一个区间的问题，其关键是找到target落在哪里的判断添加。

```
//find ele in a circular ordered array
func search3(nums []int, target int) int {
    if len(nums) == 0 {
        return -1
    }

    head, tail := nums[0], nums[len(nums)-1]
    start, end := 0, len(nums)-1
    for start <= end {
        mid := start + (end-start)>>1
        if nums[mid] == target {
            return mid
        }

        if nums[mid] > target {
            //may be on smaller part
            if nums[mid] > tail && target <= tail {
                start = mid + 1
            } else {
                end = mid - 1
            }
        } else {
            //may be on bigger part
            if nums[mid] < head && target >= head {
                end = mid - 1
            } else {
                start = mid + 1
            }
        }
    }

    return -1
}

```
