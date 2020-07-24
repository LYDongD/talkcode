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

