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

