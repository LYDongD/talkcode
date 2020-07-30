## kafka 源码分析

####  环境准备

> 需要准备哪些环境

* 环境 = 源码 + idea + 语言环境 + 构建工具
    * 源代码：kafka主分支
    * 语言：jdk-8 + scala 2.13
    * 构建工具: gradle(6.5) -> gradlew
    * idea:
	* 安装scalla插件 -> 指定scala的sdk（2.13)
	* 指定gradle构建方式（从wrapper.properties)

* 验证环境
    * 使用gradlew build 构建kafka项目，查看是否可以构建成功
	* gradlew build -x test 可跳过测试
	* 为什么使用gradlew
	    * gradlew = gradle wrapper，用gradlew构建意味着用项目指定version的gradle进行构建，避免出现兼容性问题
 
 > 模块分析

* 模块分析 = 模块功能 + 模块关系
    * core(broker) 
	* controller
	* coordinate
	* log
	* network
	* server
    * client
    * connector
    * stream
    * bin

这里面core是核心模块，其他模块要么依赖于core，要么需要已core进行交互

#### scala 语法热身

| 语言 | 变量 | 集合类型 | 条件 | 循环 | 函数 | 类与对象 | 
| :--: | :--: | :--: | :--: | :--: | :--: | :--: |
| scala | val/var a = 1 | List/Map/Set/Turple/Option/Iterator  | if (a > 7) println("big") else println("small") | var i = 0 for (i <- 1 to 10) { print(i)} | def main(args : Array[String]): Unit = {} | class/trait/object/case-match |
| shell | a = 1；$a | 字典：declare -A dic; dic=([k1]="v1" [k2]="v2"); echo ${dic["k1"]} ${dic["k2"]} 数组：b=(1,2,"sd"); echo ${b[0]} |  if [ $1 == 'node1' ]; then xxxxx fi | while true; do xxxx  done  | function a() {echo get parameter $1}; a "hello"| 不能定义对象
| go | a := 1 | a := []int{}, b:=make(map[int]int) | if a == 3 {} else {} | for i := 1; i < 10; i++ {} ; for i, ele := range list {} | func socre(a string) (int, string)  | type struct CustomObject{} |


#### 日志模块

##### 日志段：保存消息文件的对象是怎么实现的？

1. 消息 -> 分区 -> 日志（日志目录) -> 日志段 -> 消息文件, 位移索引文件，时间戳索引文件，事务索引文件
    * 消息是以日志段为单位进行操作的
        * 日志段操作 -> 写，读和启动时恢复
    * 分区越多，日志段越多，生成的索引文件越多
        * 可能导致fd等系统软资源饱和
        * broker启动恢复时，需要加载消息文件并重建索引，文件很多导致启动变慢
        * 日志分段如果同时滚动分隔，会产生大量IO，可能导致io使用率饱和
    * 读写方法中的参数基本可以通过producer和consumer client 参数进行设置，例如单次消费最大字节数

2. broker启动 -> 读取消息文件并重建索引
    * 如果消息文件在写入时发生网络抖动或宕机将导致消息文件有效字节数 < 文件字节数（validBytes < logBytes)
        * 恢复时会有一个truncate操作, 将消息文件截断到有效字节位置
        * 如果validBytes > logBytes -> 抛出异常，终止截断操作 （思考题）


#### 日志：日字段是如何被加载的？

* 什么是高水位，LEO（log end offset)和 leader epoch ？它们的作用以及更新机制是什么？
    * 高水位 （水位通常是时间戳或偏移量，是状态的临界点）
        * 指向第一条未提交的消息offset
        * 只有提交的消息才是可见（可被消费）的消息
        * 高水位是分区副本协商的同步临界值
        * hw = max(leader.current_hw, follower.min(leo_1, leo_2...leo_n)
            * leader所在节点会保存其他所有远程副本的HW和LEO，这里的leo是其他follower副本的leo, 其目的是在更新hw时与这些LEO进行协商确定一个最新的HW

    * log end offset 
        * 指向下一条消息写入offert
        * 当消息写入磁盘后，更新当前副本的LEO
        * HW <= LEO
    * leader epoch （<epoch, startOffset>)
        * leader 版本以及基于该版本的第一条消息偏移量
        * 当新leader写入新的消息时，会生成新的epoch元祖
        * 防止leader重新选举后导致消息丢失的问题
            * 重新选举后，身份互换，follower可能领先于leader，此时follower需要进行消息截断
                * 截断的依据是<epoch, startOffset>, 如果leader的epoch比自己大，则截断到对应的startOffset，从而保证数据一致。
        * leader重新选举后，epoch + 1

综上，我们可以得出以下几个结论：

1. 生产者消息写入磁盘后（ack完成），该消息未必是可见的，因为hw未必更新到当前LEO
2. 如果仅仅依靠HW + LEO 依然无法保证高可用（仍然有数据丢失的风险），需要通过epoch来防止数据丢失
3. 副本同步 = 数据pull + HW + LEO，只有完成同步并被确认的消息才是可见的

* 数据副本同步流程
    * leader副本消息写入磁盘，更新LEO (HW=0, LEO=1)
    * follower根据当前offset(offset=0)拉取消息，写入磁盘, 更新LEO(HW=0, LEO=1)
    * follower再次拉取消息，此时offset=1
        * leader根据follower offset更新当前节点的远程副本LEO=1
        * leader更新hw = max(current_hw, min(leo)) = 1
        * leader返回最新hw, follower收到消息后更新副本hw
    * 经过两次消息拉取后，leader和follower的高水位一致（最终一致性）
        *  中间过程会出现hw错配的情况（不一致）
            * follower 宕机 -> 恢复后重新拉取leader的leo，只有该leo比自己的leo小才截断日志（避免丢数据）
            * leader 宕机 -> follower被选举为新的leader，新消息写入时更新epoch++
                * 恢复后重新拉取新leader的LEO并判断是否需要截断
        

* 思考题：Log中maybeIncrementHighWatermark的实现原理是什么？

该方法用于更新当前日志的高水位(递增），当前仅当以下2个条件任意一个满足时才执行：

1. 新高水位 ∈ (旧高水位，LEO]
2. 新 = 旧 && 新水位在新的日志分段上

当follower来拉取（fetch)消息时，leader根据其提交的位移(offset)计算出新的高水位并进行更新。
另外，该方法加锁，保证线程互斥。

#### 日志索引

> 如何重设位移？

重设位移 = 7 + 2 , 即7种策略和2个方法。7种策略可以分为时间维度和位移维度的重设，时间策略包括恢复到指定时间和一段时间之前；位移策略包括恢复到起始，最新，当前，指定位移和相对当前指定偏移的位置。我们可以通过kafka工具脚本或api完成位移重设。

以java api为例，例如我们想补偿消费过去18h的消息，在不影响现有消费的情况下，可以启动一个新的消费组，重设该消费组的位移后进行消费补偿：


duration -> offsetsForTimes -> seek -> poll -> filter -> process

核心api:

Map<TopicPartition, OffsetAndTimestamp> Consumer#offsetsForTimes(Map<TopicPartition, Long>) -> 根据时间拉取对应的位移

Map<TopicPartition, OffsetAndTimestamp> Consumers#seek(topicPartition, offset) -> 重设指定分区的位移

拉取消息并处理

```
while(true) {
    ConsumerRecords<String, byte[]> records = consumer.poll(1000);
    filter(records);
    process(records);
    consumer.commitAsync()
}

```


