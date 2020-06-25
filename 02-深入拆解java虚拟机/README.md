## 深入拆解java虚拟机

#### [06 | JVM是如何处理异常的？](https://time.geekbang.org/column/article/12134)

> jvm如何处理异常？

* 代码层面如何处理异常？
    * 异常抛出
        * 自动抛出
        * 手动抛出 -> throw 关键字
    * 异常捕获
        * try-catch-finally
        * try-catch-with-resource
            * 自动关闭资源
            * 不会覆盖finally中新发生的异常

* jvm 如何处理异常？
    * jvm主要解决执行指令时，发生异常，如何跳转的问题
        * 每个方法都维护一个异常表，记录所有异常监控条目
            * try：from -> to 
            * catch: target
            * 各种声明catch的Exception： 异常类型
            * any: 未catch异常
            
        * 发生异常时，异常处理器遍历异常表，根据发生异常的字节码索引，找到对应的条目
            * 遍历查找 -> 类型匹配 -> 指令跳转（target)
            * 匹配到any时（没有catch对应类型的异常), 向上抛出异常
                * 如果有finally，执行finally相关指令
                * 方法退出（弹栈）
                * 在下一层栈帧（上个方法）中执行搜索

        * 发送异常时，异常处理器需要生成当前栈帧的快照，并创建异常对象
            * 该操作比较耗时，增加方法执行的延时
                * 避免把异常当做已知的条件分支逻辑

* demo

1 实现异常捕获代码
2 编译并反编译查看异常表
3 梳理发生异常或未发生异常时的指令执行顺序

```
public void exceptionHandle(){
    try {
        tryBlock = 1;
    }catch (ArrayIndexOutOfBoundsException ae){
        catchBlock = 2;
    }finally {
        finallyBlock = 3;
    }
}

```
javap -v xxx.class

```
Exception table:
   from    to  target type
       0     8    19   Class java/lang/ArrayIndexOutOfBoundsException
       0     8    39   any
      19    28    39   any
```           

可以看到虚拟机通过exception table 管理异常监控条目，发生异常时，指令跳转到19执行catch block或39去执行finally block对应的指令

```
19: astore_1  //写入locals[1] (局部变量表），将操作数栈的计算结果写入locals数组
20: aload_0   //读locals[0]的引用(a代表引用), this指针
21: iconst_2  //压入操作数栈：常量2
22: invokestatic  #2  //调用Integer.ValueOf(2), 去常量池中找方法引用                
25: putfield      #6  //更新成员变量tryBlock, 去常量池中找成员变量引用                
28: aload_0   //读locals[0]的引用， this指针
29: iconst_3 //压入操作数栈：常量3
30: invokestatic  #2 //调用Integer.ValueOf(3), 去常量池中找方法引用               
33: putfield      #4 //赋值给成员变量finallyBlock, 去常量池中找成员变量引用                

Constant pool ：

Constant pool:
   #1 = Methodref          #8.#23         // java/lang/Object."<init>":()V
   #2 = Methodref          #24.#25        // java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
   #3 = Fieldref           #7.#26         // com/example/demo/exception/ExceptionDemo.tryBlock:Ljava/lang/Integer;
   #4 = Fieldref           #7.#27         // com/example/demo/exception/ExceptionDemo.finallyBlock:Ljava/lang/Integer;
   #5 = Class              #28            // java/lang/ArrayIndexOutOfBoundsException
   #6 = Fieldref           #7.#29         // com/example/demo/exception/ExceptionDemo.catchBlock:Ljava/lang/Integer;


```
