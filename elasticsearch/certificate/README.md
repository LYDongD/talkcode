## ES考证之路

#### cross cluster search 跨集群搜索

* 节点2启动kibana后es进程退出
    * 重新启动es失败，无异常日志
    * 查看系统错误日志：dmesg -T
        * 发现有java进程发生OOM，说明es进程因为oom被系统杀死


