## 副本管理模块

#### 请简单描述一下 handlePartitionsWithErrors 方法的实现原理

> 什么情况下会调用该方法

* mayFetch, 拉取leader副本消息时出错
* truncateToEpochEndOffsets， 截断当前分区到epoch offset时出错
* processFetchRequest，处理副本数据拉取请求时出错

> 方法流程

1. 打印错误分区及产生分区错误的方法
2. 遍历错误分区，如果分区的状态不是延时分区，则更新分区状态(PartitionFetchState)的延时以及其他元信息
    * PartitionStates 用 LinkedHashMap<TopicPartition,S> 保存分区状态
    * 分区状态为PartitionFetchState
        * isReadyForFetch -> 副本状态为Fetching且没有延时, 当前状态为活跃状态
        * isTruncating -> 副本状态为Truncating 且 没有延时，例如leader降级为follower需要truncate
        * isDelayed -> 当前有延时（DelayedItem不为空且延时时间未到）, 分区错误时设置
3. 唤醒其他等待partitionMapCond条件的线程
    * mayFetch, 无可用分区时，等待
    * processFetchRequest，请求leader拉取数据时抛出异常，等待，等待时间未错误分区的延时

